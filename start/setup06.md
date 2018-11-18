#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第6步，网络初始化（`src/init.cpp::AppInitMain()`）

1.  生成智能指针对象 g_connman，类型为 `CConnman`。

        g_connman = std::unique_ptr<CConnman>(new CConnman(GetRand(std::numeric_limits<uint64_t>::max()), GetRand(std::numeric_limits<uint64_t>::max())));
        CConnman& connman = *g_connman;

2.  生成智能指针对象 peerLogic，类型为 `PeerLogicValidation`。

        peerLogic.reset(new PeerLogicValidation(&connman, scheduler, gArgs.GetBoolArg("-enablebip61", DEFAULT_ENABLE_BIP61)));

    PeerLogicValidation 继承了 CValidationInterface、NetEventsInterface 两个类。实现 CValidationInterface 这个类可以订阅验证过程中产生的事件。实现 NetEventsInterface 这个类可以处理消息网络消息。

3.  注册各种验证处理器，即信号处理器，在发送信号时会调用这些处理器。

        RegisterValidationInterface(peerLogic.get());

    方法具体实现如下：

        void RegisterValidationInterface(CValidationInterface* pwalletIn) {
            g_signals.m_internals->UpdatedBlockTip.connect(boost::bind(&CValidationInterface::UpdatedBlockTip, pwalletIn, _1, _2, _3));
            g_signals.m_internals->TransactionAddedToMempool.connect(boost::bind(&CValidationInterface::TransactionAddedToMempool, pwalletIn, _1));
            g_signals.m_internals->BlockConnected.connect(boost::bind(&CValidationInterface::BlockConnected, pwalletIn, _1, _2, _3));
            g_signals.m_internals->BlockDisconnected.connect(boost::bind(&CValidationInterface::BlockDisconnected, pwalletIn, _1));
            g_signals.m_internals->TransactionRemovedFromMempool.connect(boost::bind(&CValidationInterface::TransactionRemovedFromMempool, pwalletIn, _1));
            g_signals.m_internals->ChainStateFlushed.connect(boost::bind(&CValidationInterface::ChainStateFlushed, pwalletIn, _1));
            g_signals.m_internals->Broadcast.connect(boost::bind(&CValidationInterface::ResendWalletTransactions, pwalletIn, _1, _2));
            g_signals.m_internals->BlockChecked.connect(boost::bind(&CValidationInterface::BlockChecked, pwalletIn, _1, _2));
            g_signals.m_internals->NewPoWValidBlock.connect(boost::bind(&CValidationInterface::NewPoWValidBlock, pwalletIn, _1, _2));
        }

    静态变量 g_signals 在程序启动前生成，m_internals 在第4a 步应用程序初始化过程中生成。

4.  根据命令行参数 `-uacomment`，处理追加到用户代理的字符串。

        std::vector<std::string> uacomments;
        for (const std::string& cmt : gArgs.GetArgs("-uacomment")) {
            if (cmt != SanitizeString(cmt, SAFE_CHARS_UA_COMMENT))
                return InitError(strprintf(_("User Agent comment (%s) contains unsafe characters."), cmt));
            uacomments.push_back(cmt);
        }

5.  构造并检查版本字符串长度是否大于 `version` 消息中版本的最大长度。

        strSubVersion = FormatSubVersion(CLIENT_NAME, CLIENT_VERSION, uacomments);
        if (strSubVersion.size() > MAX_SUBVERSION_LENGTH) {
            return InitError(strprintf(_("Total length of network version string (%i) exceeds maximum length (%i). Reduce the number or size of uacomments."),
                strSubVersion.size(), MAX_SUBVERSION_LENGTH));
        }

6.  如果指定了 `onlynet` 参数，则设置可以对接进行连接的类型，比如：ipv4、ipv6、onion。

        if (gArgs.IsArgSet("-onlynet")) {
            std::set<enum Network> nets;
            for (const std::string& snet : gArgs.GetArgs("-onlynet")) {
                enum Network net = ParseNetwork(snet);
                if (net == NET_UNROUTABLE)
                    return InitError(strprintf(_("Unknown network specified in -onlynet: '%s'"), snet));
                nets.insert(net);
            }
            for (int n = 0; n < NET_MAX; n++) {
                enum Network net = (enum Network)n;
                if (!nets.count(net))
                    SetLimited(net);
            }
        }

    上面的代码首先把 `-onlynet` 参数指定的只允许对外连接的网络类型加入集合中，然后进行 for 遍历，如果当前的类型不在允许的集合中，则调用 `SetLimited` 方法，设置这些类型为受限的。

7.  获取是否允许进行 DNS 查找，是否进行代理随机

        fNameLookup = gArgs.GetBoolArg("-dns", DEFAULT_NAME_LOOKUP);
        bool proxyRandomize = gArgs.GetBoolArg("-proxyrandomize", DEFAULT_PROXYRANDOMIZE);

    两者默认都为真。

8.  处理网络代理。

    如果指定了 `-proxy`，且不等于 0，即指定了代理地址，进行下面的处理：

    -   调用 `Lookup` 方法，根据指定的代理，通过 DNS查找，发现代理服务器的地址。

    -   生成 proxyType 对象。

    -   设置 IPv4、IPv6、Tor 网络的代理。

    -   设置命名（域名）代理。

    -   设置不限制连接到 Tor 网络。

    具体代码如下：

        std::string proxyArg = gArgs.GetArg("-proxy", "");
        SetLimited(NET_ONION);
        if (proxyArg != "" && proxyArg != "0") {
            CService proxyAddr;
            if (!Lookup(proxyArg.c_str(), proxyAddr, 9050, fNameLookup)) {
                return InitError(strprintf(_("Invalid -proxy address or hostname: '%s'"), proxyArg));
            }

            proxyType addrProxy = proxyType(proxyAddr, proxyRandomize);
            if (!addrProxy.IsValid())
                return InitError(strprintf(_("Invalid -proxy address or hostname: '%s'"), proxyArg));

            SetProxy(NET_IPV4, addrProxy);
            SetProxy(NET_IPV6, addrProxy);
            SetProxy(NET_ONION, addrProxy);
            SetNameProxy(addrProxy);
            SetLimited(NET_ONION, false); // by default, -proxy sets onion as reachable, unless -noonion later
        }

9.  处理洋葱网络。 如果指定了 `onion` 参数，则处理洋葱网络的相关设置。

    如果指定了 `-onion`，且不等于空字符串，即指定了洋葱代理地址，进行下面的处理：

    -   如果参数等于 0，设置洋葱网络受限，即不可达。否则，进行下面的处理。

    -   调用 `Lookup` 方法，根据指定的代理，通过 DNS查找，发现代理服务器的地址。

    -   生成 proxyType 对象。

    -   设置 Tor 网络的代理。

    -   设置不限制连接到 Tor 网络。

    具体代码如下：

        std::string onionArg = gArgs.GetArg("-onion", "");
        if (onionArg != "") {
            if (onionArg == "0") { // Handle -noonion/-onion=0
                SetLimited(NET_ONION); // set onions as unreachable
            } else {
                CService onionProxy;
                if (!Lookup(onionArg.c_str(), onionProxy, 9050, fNameLookup)) {
                    return InitError(strprintf(_("Invalid -onion address or hostname: '%s'"), onionArg));
                }
                proxyType addrOnion = proxyType(onionProxy, proxyRandomize);
                if (!addrOnion.IsValid())
                    return InitError(strprintf(_("Invalid -onion address or hostname: '%s'"), onionArg));
                SetProxy(NET_ONION, addrOnion);
                SetLimited(NET_ONION, false);
            }
        }

10.  处理通过 `-externalip` 参数设置的外部 IP地址。

    获取并遍历所有指定的外部地址，进行如下处理：调用 `Lookup` 方法进行DNS 查找。如果成功则调用 `AddLocal` 方法，保存新的地址。否则，抛出初始化错误。

            for (const std::string& strAddr : gArgs.GetArgs("-externalip")) {
                CService addrLocal;
                if (Lookup(strAddr.c_str(), addrLocal, GetListenPort(), fNameLookup) && addrLocal.IsValid())
                    AddLocal(addrLocal, LOCAL_MANUAL);
                else
                    return InitError(ResolveErrMsg("externalip", strAddr));
            }

11.  如果设置了 `maxuploadtarget` 参数，则设置最大出站限制。

            if (gArgs.IsArgSet("-maxuploadtarget")) {
                nMaxOutboundLimit = gArgs.GetArg("-maxuploadtarget", DEFAULT_MAX_UPLOAD_TARGET)*1024*1024;
            }

