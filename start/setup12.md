#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第12步，启动节点（`src/init.cpp::AppInitMain()`）

1.  获取活跃区块链的当前调度。

        chain_active_height = chainActive.Height();

2.  如果指定了监听洋葱网络 `-listenonion`，调用 `StartTorControl` 函数，开始 Tor 控制。

    代码如下所示：

        void StartTorControl()
        {
            assert(!gBase);
        #ifdef WIN32
            evthread_use_windows_threads();
        #else
            evthread_use_pthreads();
        #endif
            gBase = event_base_new();
            if (!gBase) {
                LogPrintf("tor: Unable to create event_base\n");
                return;
            }

            torControlThread = std::thread(std::bind(&TraceThread<void (*)()>, "torcontrol", &TorControlThread));
        }

    libevent默认情况下是单线程，每个线程有且仅有一个event_base。为了保存多线程下是安全的，首先需要调用 `evthread_use_pthreads` 、`evthread_use_windows_threads` 等两个方法，前面是 linux 下的，后面是 windows 下的。

    在处理完多线程设置后，调用 `event_base_new` 方法，创建一个默认的 event_base。

    最后，启动一个 Tor 控制线程。具体调用 `std::thread` 方法，创建一个线程，线程的具体执行方法为 `std::bind` 返回的绑定函数。标准绑定函数的第一个参数为要执行的函数，此处为 `TraceThread`，第二个参数为线程的名字 `torcontrol`，第三个参数为线程要执行的真正方法，此处为 `TorControlThread` 函数，后面两个参数都会做为参数，传递到第一个函数。

    `TraceThread` 函数，调用 `RenameThread` 方法，把线程名字设置为 `bitcoin-torcontrol`，然后执行传递进来的 `TorControlThread` 函数。后者会生成一个 Tor 控制器，然后调用 `event_base_dispatch` 方法，分发事件。代码如下：

        static void TorControlThread()
        {
            TorController ctrl(gBase, gArgs.GetArg("-torcontrol", DEFAULT_TOR_CONTROL));

            event_base_dispatch(gBase);
        }

    `TorController` 构造函数中会做几件重要的事情：

    -   首先，调用 `event_new` 方法生成一个 event 对象，event 对象的回调函数为 `reconnect_cb` 。

    -   然后，调用 `TorControlConnection::Connect` 方法连接到 Tor 控制器。

        这个方法又会做几件事情：

        -   解析 Tor 控制器的地址。

        -   调用 `bufferevent_socket_new` 方法，基于套接字生成一个 bufferevent。

        -   设置 bufferevent 的回调方法，包括：读取回调函数为 `TorControlConnection::readcb`，写入回调函数为空，事件回调函数为 `TorControlConnection::eventcb`，同时指定 bufferevent 启用读写标志。

        -   设置 `TorControlConnection` 连接、断开连接的两个指针函数分别为：`TorController::connected_cb` 和 `TorController::disconnected_cb`。

        -   调用 `bufferevent_socket_connect` 方法，连接到前面生成的 bufferevent。

            方法在连接成功后，会立即调用事件回调函数 `TorControlConnection::eventcb`。

3.  调用 `Discover` 函数，开始发现本节点的地址。

    方法内首先判断是否已经处理过。如果没有，那么开始发现本节点的地址。具体处理分为 windows 和 linux，下面主要讲述 linux 下的处理。

    调用 `getifaddrs` 方法，查找系统所有的网络接口的信息，包括以太网卡接口和回环接口等。本方法返回一个如下的结构体：

        struct ifaddrs   
        {   
            struct ifaddrs  *ifa_next;    /* 列表中的下一个条目 */   
            char            *ifa_name;    /* 接口的名称 */   
            unsigned int     ifa_flags;   /* 来自 SIOCGIFFLAGS 的标志 */   
            struct sockaddr *ifa_addr;    /* 接口的地址 */   
            struct sockaddr *ifa_netmask; /* 接口的网络掩码 */   
            union   
            {   
                struct sockaddr *ifu_broadaddr; /* 接口的广播地址 */   
                struct sockaddr *ifu_dstaddr; /* 点对点的目标地址 */   
            } ifa_ifu;   
            #define              ifa_broadaddr ifa_ifu.ifu_broadaddr   
            #define              ifa_dstaddr   ifa_ifu.ifu_dstaddr   
            void            *ifa_data;    /* Address-specific data */   
        };   

    如果可以获取接口信息，则遍历每一个接口，进行如下处理：

    -   如果接口地址为空，则处理下一个。

    -   如果不是接口标志不是 IFF_UP ，则处理下一个。

    -   如果接口名称是 lo 或 lo0，则处理下一个。

    -   如果接口是 tcp,TCP 等，则生成 IP 地址对象，然后调用 `AddLocal` 方法，保存本地地址。

    -   如果接口是 IPV6，则则生成 IP 地址对象，然后调用 `AddLocal` 方法，保存本地地址。

    代码如下所示：

        if (getifaddrs(&myaddrs) == 0)
        {
            for (struct ifaddrs* ifa = myaddrs; ifa != nullptr; ifa = ifa->ifa_next)
            {
                if (ifa->ifa_addr == nullptr) continue;
                if ((ifa->ifa_flags & IFF_UP) == 0) continue;
                if (strcmp(ifa->ifa_name, "lo") == 0) continue;
                if (strcmp(ifa->ifa_name, "lo0") == 0) continue;
                if (ifa->ifa_addr->sa_family == AF_INET)
                {
                    struct sockaddr_in* s4 = (struct sockaddr_in*)(ifa->ifa_addr);
                    CNetAddr addr(s4->sin_addr);
                    if (AddLocal(addr, LOCAL_IF))
                        LogPrintf("%s: IPv4 %s: %s\n", __func__, ifa->ifa_name, addr.ToString());
                }
                else if (ifa->ifa_addr->sa_family == AF_INET6)
                {
                    struct sockaddr_in6* s6 = (struct sockaddr_in6*)(ifa->ifa_addr);
                    CNetAddr addr(s6->sin6_addr);
                    if (AddLocal(addr, LOCAL_IF))
                        LogPrintf("%s: IPv6 %s: %s\n", __func__, ifa->ifa_name, addr.ToString());
                }
            }
            freeifaddrs(myaddrs);
        }

4.  如果指定了 `upnp` 参数，则调用 `StartMapPort` 函数，开始进行端口映射。

        if (gArgs.GetBoolArg("-upnp", DEFAULT_UPNP)) {
            StartMapPort();
        }

5.  生成选项对象，并进行初始化。

        CConnman::Options connOptions;
        connOptions.nLocalServices = nLocalServices;
        connOptions.nMaxConnections = nMaxConnections;
        connOptions.nMaxOutbound = std::min(MAX_OUTBOUND_CONNECTIONS, connOptions.nMaxConnections);
        connOptions.nMaxAddnode = MAX_ADDNODE_CONNECTIONS;
        connOptions.nMaxFeeler = 1;
        connOptions.nBestHeight = chain_active_height;
        connOptions.uiInterface = &uiInterface;
        connOptions.m_msgproc = peerLogic.get();
        connOptions.nSendBufferMaxSize = 1000*gArgs.GetArg("-maxsendbuffer", DEFAULT_MAXSENDBUFFER);
        connOptions.nReceiveFloodSize = 1000*gArgs.GetArg("-maxreceivebuffer", DEFAULT_MAXRECEIVEBUFFER);
        connOptions.m_added_nodes = gArgs.GetArgs("-addnode");

        connOptions.nMaxOutboundTimeframe = nMaxOutboundTimeframe;
        connOptions.nMaxOutboundLimit = nMaxOutboundLimit;

    上面的代码基本就是设置本地支持的服务、最大连接数、最大出站数、最大节点数、最大费率、活跃区块链的高度、节点逻辑验证器、发送的最大缓冲值、接收的最大缓冲值、连接的节点数等。

6.  如果指定了 `-bind` 参数，则处理绑定参数。

        for (const std::string& strBind : gArgs.GetArgs("-bind")) {
            CService addrBind;
            if (!Lookup(strBind.c_str(), addrBind, GetListenPort(), false)) {
                return InitError(ResolveErrMsg("bind", strBind));
            }
            connOptions.vBinds.push_back(addrBind);
        }

    遍历所有的绑定地址，调用 `Lookup` 方法，进行 DNS查找。如果可以找到对应 IP地址，把生成的 `CService` 对象放入选项对象的 `vBinds` 属性中。

7.  如果指定了 `-whitebind` 参数，则处理绑定参数。

        for (const std::string& strBind : gArgs.GetArgs("-whitebind")) {
            CService addrBind;
            if (!Lookup(strBind.c_str(), addrBind, 0, false)) {
                return InitError(ResolveErrMsg("whitebind", strBind));
            }
            if (addrBind.GetPort() == 0) {
                return InitError(strprintf(_("Need to specify a port with -whitebind: '%s'"), strBind));
            }
            connOptions.vWhiteBinds.push_back(addrBind);
        }

    遍历所有的绑定地址，调用 `Lookup` 方法，进行 DNS查找。如果可以找到对应 IP地址，且对应的端口号不等于0，把生成的 `CService` 对象放入选项对象的 `vWhiteBinds` 属性中。

8.  如果指定了 `-whitelist` 参数，则处理白名单列表。

        for (const auto& net : gArgs.GetArgs("-whitelist")) {
            CSubNet subnet;
            LookupSubNet(net.c_str(), subnet);
            if (!subnet.IsValid())
                return InitError(strprintf(_("Invalid netmask specified in -whitelist: '%s'"), net));
            connOptions.vWhitelistedRange.push_back(subnet);
        }

    遍历白名单列表，调用 `LookupSubNet` 方法，查找对应的子网掩码，如果对应的子网掩码是有效的，那么放入选项对象的 `vWhitelistedRange` 属性中。

9.  取得参数 `seednode` 指定的值，放入选项对象的 `vSeedNodes` 属性中。

        connOptions.vSeedNodes = gArgs.GetArgs("-seednode");

10.  调用 `CConnman` 对象的 `Start` 方法，初始所有的出站连接。

    **本方法非常非常重要，因为它启动了一个重要的流程，即底层的 P2P 网络建立和消息处理流程**。

    具体分析如下：

    -   调用 `Init` 方法，根据选项对象设置对象的属性，包括：本地支持的服务、最大连接数、最大出站数、最大增加的节点数、最大费率、最佳区块链高度等等。不细说，代码如下：

                void Init(const Options& connOptions) {
                    nLocalServices = connOptions.nLocalServices;
                    nMaxConnections = connOptions.nMaxConnections;
                    nMaxOutbound = std::min(connOptions.nMaxOutbound, connOptions.nMaxConnections);
                    nMaxAddnode = connOptions.nMaxAddnode;
                    nMaxFeeler = connOptions.nMaxFeeler;
                    nBestHeight = connOptions.nBestHeight;
                    clientInterface = connOptions.uiInterface;
                    m_msgproc = connOptions.m_msgproc;
                    nSendBufferMaxSize = connOptions.nSendBufferMaxSize;
                    nReceiveFloodSize = connOptions.nReceiveFloodSize;
                    {
                        LOCK(cs_totalBytesSent);
                        nMaxOutboundTimeframe = connOptions.nMaxOutboundTimeframe;
                        nMaxOutboundLimit = connOptions.nMaxOutboundLimit;
                    }
                    vWhitelistedRange = connOptions.vWhitelistedRange;
                    {
                        LOCK(cs_vAddedNodes);
                        vAddedNodes = connOptions.m_added_nodes;
                    }
                }

    -   接下来，使用锁初始一些比较重要的属性，包括：设置总接收的字节 `nTotalBytesRecv`、总的发送数量`nTotalBytesSent`、`nMaxOutboundTotalBytesSentInCycle`、`nMaxOutboundCycleStartTime` 等都为0。

                {
                    LOCK(cs_totalBytesRecv);
                    nTotalBytesRecv = 0;
                }
                {
                    LOCK(cs_totalBytesSent);
                    nTotalBytesSent = 0;
                    nMaxOutboundTotalBytesSentInCycle = 0;
                    nMaxOutboundCycleStartTime = 0;
                }

    -   再接下来，获取节点绑定的本地地址和端口，并生成对应的套接字，接受别的节点的请求。

                if (fListen && !InitBinds(connOptions.vBinds, connOptions.vWhiteBinds)) {
                    if (clientInterface) {
                        clientInterface->ThreadSafeMessageBox(
                            _("Failed to listen on any port. Use -listen=0 if you want this."),
                            "", CClientUIInterface::MSG_ERROR);
                    }
                    return false;
                }

        `InitBinds` 方法，接收 `-bind` 和 `-whitebind` 参数生成的集合，并解析各个地址，生成套接字，并进行监听。具体分析如下：

        -   首先，处理`-bind` 地址集合。

                    for (const auto& addrBind : binds) {
                        fBound |= Bind(addrBind, (BF_EXPLICIT | BF_REPORT_ERROR));
                    }

        -   然后，处理 `-whitebind` 地址集合。

                    for (const auto& addrBind : whiteBinds) {
                        fBound |= Bind(addrBind, (BF_EXPLICIT | BF_REPORT_ERROR | BF_WHITELIST));
                    }

        -   如果，两个参数都没有指定，则使用下面代码进行处理。

                    if (binds.empty() && whiteBinds.empty()) {
                        struct in_addr inaddr_any;
                        inaddr_any.s_addr = INADDR_ANY;
                        struct in6_addr inaddr6_any = IN6ADDR_ANY_INIT;
                        fBound |= Bind(CService(inaddr6_any, GetListenPort()), BF_NONE);
                        fBound |= Bind(CService(inaddr_any, GetListenPort()), !fBound ? BF_REPORT_ERROR : BF_NONE);
                    }

        从以上代码可以看出来，三种情况下，处理基本相同，都是调用 `Bind` 方法来处理。下面，我们进进入这个方法一控究竟。这个方法的主体是调用 `BindListenPort` 方法进行处理。下面我们开始讲解这个方法。

        -   首先，生成一个通用的网络地址 sockaddr 对象，类型为 sockaddr_storage，它的长度是 128个字节。

        -   然后，调用 `addrBind.GetSockAddr((struct sockaddr*)&sockaddr, &len)` 方法来设置网络地址 sockaddr。

            `GetSockAddr` 方法内部根据地址是 IPV4 或 IPV6，分别进行处理。

            如果是 IPV4，则生成 sockaddr_in 地址对象，然后调用 `memset` 把结构体所占内存用0填充，然后调用 `GetInAddr` 方法来设置地址对象的地址字段，最后设置地址类型为 AF_INET 和端口号。

            如果是 IPV6，则生成 sockaddr_in6 地址对象，然后调用 `memset` 把结构体所占内存用0填充，然后调用 `GetIn6Addr` 方法来设置地址对象的地址字段，最后设置地址类型为 AF_INET6 和端口号。

        -   再然后，调用 `CreateSocket(addrBind)` 方法生成套接字对象。

            方法处理如下：

            -   首先，生成一个通用的网络地址 sockaddr 对象，类型为 sockaddr_storage，然后，调用 `addrBind.GetSockAddr((struct sockaddr*)&sockaddr, &len)` 方法来设置网络地址 sockaddr。具体分析详见上面。

            -   然后，生成套接字。

                socket(((struct sockaddr*)&sockaddr)->sa_family, SOCK_STREAM, IPPROTO_TCP)

            -   再然后，对套接字进行一些检查和处理，不详述。

        -   套接字生成之后，接下来把套接字绑定到指定的地址上，并监听入站请求。

                    if (::bind(hListenSocket, (struct sockaddr*)&sockaddr, len) == SOCKET_ERROR)
                    {
                        int nErr = WSAGetLastError();
                        if (nErr == WSAEADDRINUSE)
                            strError = strprintf(_("Unable to bind to %s on this computer. %s is probably already running."), addrBind.ToString(), _(PACKAGE_NAME));
                        else
                            strError = strprintf(_("Unable to bind to %s on this computer (bind returned error %s)"), addrBind.ToString(), NetworkErrorString(nErr));
                        LogPrintf("%s\n", strError);
                        CloseSocket(hListenSocket);
                        return false;
                    }
                    LogPrintf("Bound to %s\n", addrBind.ToString());

                    // Listen for incoming connections
                    if (listen(hListenSocket, SOMAXCONN) == SOCKET_ERROR)
                    {
                        strError = strprintf(_("Error: Listening for incoming connections failed (listen returned error %s)"), NetworkErrorString(WSAGetLastError()));
                        LogPrintf("%s\n", strError);
                        CloseSocket(hListenSocket);
                        return false;
                    }

        -   最后，进行一些收尾工作。

            把套接字放入 `vhListenSocket` 集合中。如果地址是可达的，并且不是白名单中的地址，则调用 `AddLocal` 方法，加入本地地址集合中。

    -   处理完地址绑定之后，接下来处理种子节点参数指定集合。

                void CConnman::AddOneShot(const std::string& strDest)
                {
                    LOCK(cs_vOneShots);
                    vOneShots.push_back(strDest);
                }

        这个方法非常简单，把每个种子节点加入 `vOneShots` 集合。

    -   接下来，从文件数据库中加载地址列表和禁止地址列表。

                {
                    CAddrDB adb;
                    if (adb.Read(addrman))
                        LogPrintf("Loaded %i addresses from peers.dat  %dms\n", addrman.size(), GetTimeMillis() - nStart);
                    else {
                        addrman.Clear(); // Addrman can be in an inconsistent state after failure, reset it
                        LogPrintf("Invalid or missing peers.dat; recreating\n");
                        DumpAddresses();
                    }
                }

                CBanDB bandb;
                banmap_t banmap;
                if (bandb.Read(banmap)) {
                    SetBanned(banmap); // thread save setter
                    SetBannedSetDirty(false); // no need to write down, just read data
                    SweepBanned(); // sweep out unused entries

                    LogPrint(BCLog::NET, "Loaded %d banned node ips/subnets from banlist.dat  %dms\n",
                        banmap.size(), GetTimeMillis() - nStart);
                } else {
                    LogPrintf("Invalid or missing banlist.dat; recreating\n");
                    SetBannedSetDirty(true); // force write
                    DumpBanlist();
                }

        代码比较简单，一看便知，不作具体展开。

    -   **最后，重中之重的线程相关处理终于要到来了**。

        -   首先，生成套接字相关的线程，以便进行网络的接收和发送。处理方法和前面线程的类似，代码如下：

                    threadSocketHandler = std::thread(&TraceThread<std::function<void()> >, "net", std::function<void()>(std::bind(&CConnman::ThreadSocketHandler, this)));

            真正执行的方法是 `ThreadSocketHandler`，这个方法太重要了，我们留在下一课网络处理中细讲。

        -   接下来，处理 DNS 种子节点线程，处理 DNS 种子相关的逻辑。代码如下：

                    if (!gArgs.GetBoolArg("-dnsseed", true))
                        LogPrintf("DNS seeding disabled\n");
                    else
                        threadDNSAddressSeed = std::thread(&TraceThread<std::function<void()> >, "dnsseed", std::function<void()>(std::bind(&CConnman::ThreadDNSAddressSeed, this)));

            真正执行的方法是 `ThreadDNSAddressSeed`，这个方法太重要了，我们留在下一课网络处理中细讲。

        -   接下来，处理出站连接。代码如下：

                    threadOpenAddedConnections = std::thread(&TraceThread<std::function<void()> >, "addcon", std::function<void()>(std::bind(&CConnman::ThreadOpenAddedConnections, this)));

            真正执行的方法是 `ThreadOpenAddedConnections`，这个方法太重要了，我们留在下一课网络处理中细讲。

        -   接下来，处理打开连接的线程。代码如下：

                    if (connOptions.m_use_addrman_outgoing || !connOptions.m_specified_outgoing.empty())
                            threadOpenConnections = std::thread(&TraceThread<std::function<void()> >, "opencon", std::function<void()>(std::bind(&CConnman::ThreadOpenConnections, this, connOptions.m_specified_outgoing)));

            真正执行的方法是 `ThreadOpenConnections`，这个方法太重要了，我们留在下一课网络处理中细讲。

        -   最最重要的线程--处理消息的线程，隆重登场。

                    threadMessageHandler = std::thread(&TraceThread<std::function<void()> >, "msghand", std::function<void()>(std::bind(&CConnman::ThreadMessageHandler, this)));

            真正执行的方法是 `ThreadMessageHandler`，这个方法太重要了，我们留在下一课网络处理中细讲。

        -   最后，定时把节点地址和禁止列表刷新到数据库文件中。
