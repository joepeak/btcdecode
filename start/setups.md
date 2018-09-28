#   如何接入比特币网络以及原理分析

##  系统启动详述

以下为系统启动过程中重要的步骤。

### 第1步，应用初始化基本设置（`src/bitcoind.cpp`）

`AppInitBasicSetup` 函数进行基本的设置。

1.  调用 `SetupNetworking` 函数，进行网络设置。

    主要是针对 Win32 系统处理套接字，别的系统直接返回真。

2.  如果不是 WIN32 系统，进行下面的处理：

    -   如果设置 `sysperms` 参数为真，调用 `umask` 函数，设置位码为 077。

    -   调用 `registerSignalHandler` 函数，设置 `SIGTERM` 信号处理器为 `HandleSIGTERM`；`SIGINT` 为 `HandleSIGTERM`；`SIGHUP` 为 `HandleSIGHUP`。


### 第2步，应用初始参数交互设置（`src/bitcoind.cpp`）

`AppInitParameterInteraction` 函数前半部分。

1.  首先，调用 `Params` 方法，获取前面初始化的 `globalChainParams` 区块链对象。

        const CChainParams& chainparams = Params();

    根据不同的网络，chainparams 的真实类型可能是 `CMainParams`，代表主网络；或者是 `CTestNetParams`，代表测试网络；或者是 `CRegTestParams` 代表回归测试网络。

2.  检查指定的区块目录是否存。如果不存在，则返回初始化错误。

        if (!fs::is_directory(GetBlocksDir(false))) {
            return InitError(strprintf(_("Specified blocks directory \"%s\" does not exist."), gArgs.GetArg("-blocksdir", "").c_str()));
        }

3.  如果同时指定了 `prune`、`txindex`，则抛出初始化错误。

    如果指定了区块修剪 `prune`，就要禁止交易索引 `txindex`，两者不兼容，只能其一。

        if (gArgs.GetArg("-prune", 0)) {
            if (gArgs.GetBoolArg("-txindex", DEFAULT_TXINDEX))
                return InitError(_("Prune mode is incompatible with -txindex."));
        }

4.  检查是否指定了 `-bind` 或 `-whitebind` 两者之一，并且同时禁止其他节点连接（`listen`）。如果是，则抛出初始化错误。

        size_t nUserBind = gArgs.GetArgs("-bind").size() + gArgs.GetArgs("-whitebind").size();
        if (nUserBind != 0 && !gArgs.GetBoolArg("-listen", DEFAULT_LISTEN)) {
            return InitError("Cannot set -bind or -whitebind together with -listen=0");
        }

5.  确保有足够的文件符可用。

    因为在类 Unix 系统中，每个套接字都是一个文件，都需要一个文件描述符。所以要检查指定的最大连接数 `maxconnections` 是否超过系统可用限制。

        int nBind = std::max(nUserBind, size_t(1));
        nUserMaxConnections = gArgs.GetArg("-maxconnections", DEFAULT_MAX_PEER_CONNECTIONS);
        nMaxConnections = std::max(nUserMaxConnections, 0);

        nMaxConnections = std::max(std::min<int>(nMaxConnections, FD_SETSIZE - nBind - MIN_CORE_FILEDESCRIPTORS - MAX_ADDNODE_CONNECTIONS), 0);
        nFD = RaiseFileDescriptorLimit(nMaxConnections + MIN_CORE_FILEDESCRIPTORS + MAX_ADDNODE_CONNECTIONS);
        if (nFD < MIN_CORE_FILEDESCRIPTORS)
            return InitError(_("Not enough file descriptors available."));
        nMaxConnections = std::min(nFD - MIN_CORE_FILEDESCRIPTORS - MAX_ADDNODE_CONNECTIONS, nMaxConnections);

        if (nMaxConnections < nUserMaxConnections)
            InitWarning(strprintf(_("Reducing -maxconnections from %d to %d, because of system limitations."), nUserMaxConnections, nMaxConnections));


### 第3步，参数到内部标志的转换处理（`src/bitcoind.cpp`）

`AppInitParameterInteraction` 函数后半部分。

1.  处理 `debug`、`debugexclude`、`debugnet` 等参数。

    如果指定了 `-debug`，则解析每个类别是否是支持的类别。如果不支持，则输入警告消息。如果需要同时指定多个类别，可分开指定，比如要调试网络与RPC 相关的信息，则配置如下：`-debug=net -debug=rpc`。

        if (gArgs.IsArgSet("-debug")) {
            const std::vector<std::string> categories = gArgs.GetArgs("-debug");

            if (std::none_of(categories.begin(), categories.end(),
                [](std::string cat){return cat == "0" || cat == "none";})) {
                for (const auto& cat : categories) {
                    if (!g_logger->EnableCategory(cat)) {
                        InitWarning(strprintf(_("Unsupported logging category %s=%s."), "-debug", cat));
                    }
                }
            }
        }

2.  如果指定了 `-socks` 参数，则提示使用 SOCKS5

        if (gArgs.IsArgSet("-socks"))
            return InitError(_("Unsupported argument -socks found. Setting SOCKS version isn't possible anymore, only SOCKS5 proxies are supported."));

3.  如果指定了 `-tor` 参数，则提示使用 `-onion`。

        if (gArgs.GetBoolArg("-tor", false))
            return InitError(_("Unsupported argument -tor found, use -onion."));

4.  如果指定了 `-benchmark` 参数，则提示使用 `-debug=bench`。

        if (gArgs.GetBoolArg("-benchmark", false))
            InitWarning(_("Unsupported argument -benchmark ignored, use -debug=bench."));

5.  如果指定了 `-whitelistalwaysrelay` 参数，则提示使用 `-whitelistrelay`，或`-whitelistforcerelay`。

        if (gArgs.GetBoolArg("-whitelistalwaysrelay", false))
            InitWarning(_("Unsupported argument -whitelistalwaysrelay ignored, use -whitelistrelay and/or -whitelistforcerelay."));

6.  如果指定了 `-blockminsize` 参数，则提示使用 `-blockminsize`。

        if (gArgs.IsArgSet("-blockminsize"))
            InitWarning("Unsupported argument -blockminsize ignored.");

7.  根据是否指定 `-checkmempool` 参数，确定是否进行合理性检查。

    在不指定这个参数的情况下，当运行主网络和测试网络时，不进行交易池合理性检查，当运行回归测试网络时，进行合理性检查。代码如下：

        // Checkmempool and checkblockindex default to true in regtest mode

        // 当运行主网络和测试网络时，DefaultConsistencyChecks 函数返回假，导致变量 ratio 为1， 为0，从而不进行交易池设置；当运行回归测试网络时，函数返回真，从而变量 ratio 为1，从而进行交易池设置
        int ratio = std::min<int>(std::max<int>(gArgs.GetArg("-checkmempool", chainparams.DefaultConsistencyChecks() ? 1 : 0), 0), 1000000);
        if (ratio != 0) {
            mempool.setSanityCheck(1.0 / ratio);
        }

        fCheckBlockIndex = gArgs.GetBoolArg("-checkblockindex", chainparams.DefaultConsistencyChecks());

8.  设置检查点默认打开

        fCheckpointsEnabled = gArgs.GetBoolArg("-checkpoints", DEFAULT_CHECKPOINTS_ENABLED);

9.  处理 `assumevalid` 参数。

    这个参数的意思是假设有效，即如果如果这个块在链中，则假定它和它的祖先是有效的，并且可能跳过它们的脚本验证。否则会验证所有的块。

        hashAssumeValid = uint256S(gArgs.GetArg("-assumevalid", chainparams.GetConsensus().defaultAssumeValid.GetHex()));
        if (!hashAssumeValid.IsNull())
            LogPrintf("Assuming ancestors of block %s have valid signatures.\n", hashAssumeValid.GetHex());
        else
            LogPrintf("Validating signatures for all blocks.\n");

10.  根据是否指定 `minimumchainwork` 参数，确定使用默认的最小工作量还是使用用户指定的最小工作量。

            if (gArgs.IsArgSet("-minimumchainwork")) {
                const std::string minChainWorkStr = gArgs.GetArg("-minimumchainwork", "");
                if (!IsHexNumber(minChainWorkStr)) {
                    return InitError(strprintf("Invalid non-hex (%s) minimum chain work value specified", minChainWorkStr));
                }
                nMinimumChainWork = UintToArith256(uint256S(minChainWorkStr));
            } else {
                nMinimumChainWork = UintToArith256(chainparams.GetConsensus().nMinimumChainWork);
            }

11. 计算内存池/交易池限制，包括处理 `maxmempool`、`limitdescendantsize` 参数。

    前者表示最大内存池，后者表示最小内存池，如果最大值小于最小值，则抛出初始化错误。

        int64_t nMempoolSizeMax = gArgs.GetArg("-maxmempool", DEFAULT_MAX_MEMPOOL_SIZE) * 1000000;
        int64_t nMempoolSizeMin = gArgs.GetArg("-limitdescendantsize", DEFAULT_DESCENDANT_SIZE_LIMIT) * 1000 * 40;
        if (nMempoolSizeMax < 0 || nMempoolSizeMax < nMempoolSizeMin)
            return InitError(strprintf(_("-maxmempool must be at least %d MB"), std::ceil(nMempoolSizeMin / 1000000.0)));

12. 如果指定了 `incrementalrelayfee`，则进行相关处理。

    `incrementalrelayfee` 定义中继的成本费率，应用于交易池限制和 BIP 125 替换。代码如下：

        if (gArgs.IsArgSet("-incrementalrelayfee"))
        {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-incrementalrelayfee", ""), n))
                return InitError(AmountErrMsg("incrementalrelayfee", gArgs.GetArg("-incrementalrelayfee", "")));
            incrementalRelayFee = CFeeRate(n);
        }
    
13. 处理 `-par` 参数，指定脚本签名的线程数量。

    代码如下：

        if (nScriptCheckThreads <= 0)
            nScriptCheckThreads += GetNumCores();
        if (nScriptCheckThreads <= 1)
            nScriptCheckThreads = 0;
        else if (nScriptCheckThreads > MAX_SCRIPTCHECK_THREADS)
            nScriptCheckThreads = MAX_SCRIPTCHECK_THREADS;

14. 处理区块修剪参数 `-prune`。

    代码如下：

        int64_t nPruneArg = gArgs.GetArg("-prune", 0);
        if (nPruneArg < 0) {
            return InitError(_("Prune cannot be configured with a negative value."));
        
        nPruneTarget = (uint64_t) nPruneArg * 1024 * 1024;
        if (nPruneArg == 1) {  // manual pruning: -prune=1
            LogPrintf("Block pruning enabled.  Use RPC call pruneblockchain(height) to manually prune block and undo files.\n");
            nPruneTarget = std::numeric_limits<uint64_t>::max();
            fPruneMode = true;
        } else if (nPruneTarget) {
            if (nPruneTarget < MIN_DISK_SPACE_FOR_BLOCK_FILES) {
                return InitError(strprintf(_("Prune configured below the minimum of %d MiB.  Please use a higher number."), MIN_DISK_SPACE_FOR_BLOCK_FILES / 1024 / 1024));
            }
            LogPrintf("Prune configured to target %uMiB on disk for block and undo files.\n", nPruneTarget / 1024 / 1024);
            fPruneMode = true;
        }

15. 处理连接超时时间 `-timeout`。

    代码如下：

        nConnectTimeout = gArgs.GetArg("-timeout", DEFAULT_CONNECT_TIMEOUT);
        if (nConnectTimeout <= 0)
            nConnectTimeout = DEFAULT_CONNECT_TIMEOUT;

16. 处理 `-minrelaytxfee` 参数。

    对于中继、挖矿和交易创建，小于此的费用被认为是零费用。解析并计算最小中继交易费用。代码如下：

        if (gArgs.IsArgSet("-minrelaytxfee")) {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-minrelaytxfee", ""), n)) {
                return InitError(AmountErrMsg("minrelaytxfee", gArgs.GetArg("-minrelaytxfee", "")));
            }
            ::minRelayTxFee = CFeeRate(n);
        } else if (incrementalRelayFee > ::minRelayTxFee) {
            ::minRelayTxFee = incrementalRelayFee;
            LogPrintf("Increasing minrelaytxfee to %s to match incrementalrelayfee\n",::minRelayTxFee.ToString());
        }

17. 处理 `-blockmintxfee` 参数。

    设置要在块创建中包含的事务满足的最低费率，即低于这个费率，交易将不进行打包。代码如下：

        if (gArgs.IsArgSet("-blockmintxfee"))
        {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-blockmintxfee", ""), n))
                return InitError(AmountErrMsg("blockmintxfee", gArgs.GetArg("-blockmintxfee", "")));
        }

18. 处理 `-dustrelayfee` 参数。

    具体代码如下：

        if (gArgs.IsArgSet("-dustrelayfee"))
        {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-dustrelayfee", ""), n))
                return InitError(AmountErrMsg("dustrelayfee", gArgs.GetArg("-dustrelayfee", "")));
            dustRelayFee = CFeeRate(n);
        }

19. 处理 `-acceptnonstdtxn` 参数。

    这个参数代表中继和挖掘非标准交易。具体代码如下：

        fRequireStandard = !gArgs.GetBoolArg("-acceptnonstdtxn", !chainparams.RequireStandard());
        if (chainparams.RequireStandard() && !fRequireStandard)
            return InitError(strprintf("acceptnonstdtxn is not currently supported for %s chain", chainparams.NetworkIDString()));

20. 处理 `-bytespersigop` 参数。

    计算中继和挖掘中交易的 sigop 的等效字节数。

        nBytesPerSigOp = gArgs.GetArg("-bytespersigop", nBytesPerSigOp);

21. 调用钱包初始接口对象的 `ParameterInteraction` 方法，初始钱包相关的参数。本方法在 `wallet/init.cpp` 文件中。

    调用代码如下：

        if (!g_wallet_init_interface.ParameterInteraction()) return false;

    代码内部具体处理如下：

    -   检查是否禁用钱包 `-disablewallet`。如果禁用，就不会加载钱包，并且会禁用钱包 RPC，这种情况下忽略 `-wallet`。

            if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
                for (const std::string& wallet : gArgs.GetArgs("-wallet")) {
                    LogPrintf("%s: parameter interaction: -disablewallet -> ignoring -wallet=%s\n", __func__, wallet);
                }

                return true;
            }

    -   确定指定的是单一钱包还是多钱包。

            gArgs.SoftSetArg("-wallet", "");
            const bool is_multiwallet = gArgs.GetArgs("-wallet").size() > 1;

    -   处理 `-blocksonly`、`-walletbroadcast` 参数。

    -   处理 `-salvagewallet`、`-rescan` 参数。

        如果指定了在启动时试图从损坏的钱包中恢复私钥，即 `-salvagewallet` 参数，那么不能使用多个钱包。

            if (gArgs.GetBoolArg("-salvagewallet", false)) {
                if (is_multiwallet) {
                    return InitError(strprintf("%s is only allowed with a single wallet file", "-salvagewallet"));
                }

                if (gArgs.SoftSetBoolArg("-rescan", true)) {
                    LogPrintf("%s: parameter interaction: -salvagewallet=1 -> setting -rescan=1\n", __func__);
                }
            }

    -   处理 `-zapwallettxes`、`-persistmempool` 参数。

        `-zapwallettxes` 参数表示删除所有钱包交易，并且仅在启动时通过 `-rescan` 恢复区块链相关的那些部分（1 =保留tx元数据，例如账户所有者和支付请求信息，2 =丢弃tx元数据）。它暗示了在启动时删除交易池中的那些交易。同时，它也暗示了进行区块链扫描，即不能是多钱包。

            bool zapwallettxes = gArgs.GetBoolArg("-zapwallettxes", false);
            if (zapwallettxes && gArgs.SoftSetBoolArg("-persistmempool", false)) {
                LogPrintf("%s: parameter interaction: -zapwallettxes enabled -> setting -persistmempool=0\n", __func__);
            }

            if (zapwallettxes) {
                if (is_multiwallet) {
                    return InitError(strprintf("%s is only allowed with a single wallet file", "-zapwallettxes"));
                }
                if (gArgs.SoftSetBoolArg("-rescan", true)) {
                    LogPrintf("%s: parameter interaction: -zapwallettxes enabled -> setting -rescan=1\n", __func__);
                }
            }      

    -   检查是否指定了 `-upgradewallet` 参数。

        在多钱包情况下，不能进行钱包升级。

            if (is_multiwallet) {
                if (gArgs.GetBoolArg("-upgradewallet", false)) {
                    return InitError(strprintf("%s is only allowed with a single wallet file", "-upgradewallet"));
                }
            }

    -   检查是否指定了 `-sysperms` 参数。

        如果指定了 `-sysperms` 参数，则抛出初始异常错误。这个参数表示了用系统默认的权限创建一个新文件，而不是 `077`，只有在禁用钱包功能的情况下才有效。

            if (gArgs.GetBoolArg("-sysperms", false))
                return InitError("-sysperms is not allowed in combination with enabled wallet functionality");

    -   检查是否同时指定了 `-prune`、`-rescan` 参数。

        在指定了修剪模式的情况下，不能执行扫描区块链的动作。所以如果同时指定了这两个参数，抛出错误。

            if (gArgs.GetArg("-prune", 0) && gArgs.GetBoolArg("-rescan", false))
                return InitError(_("Rescans are not possible in pruned mode. You will need to use -reindex which will download the whole blockchain again."));

    -   处理 `-maxtxfee` 参数。

        `-maxtxfee` 参数表示了在单个钱包的交易或原始交易中使用的最高总费用。如果设置过小，可能会中止大型交易。这个值不能小于最小中继交易费用。

            if (gArgs.IsArgSet("-maxtxfee"))
            {
                CAmount nMaxFee = 0;
                if (!ParseMoney(gArgs.GetArg("-maxtxfee", ""), nMaxFee))
                    return InitError(AmountErrMsg("maxtxfee", gArgs.GetArg("-maxtxfee", "")));
                if (nMaxFee > HIGH_MAX_TX_FEE)
                    InitWarning(_("-maxtxfee is set very high! Fees this large could be paid on a single transaction."));
                maxTxFee = nMaxFee;
                if (CFeeRate(maxTxFee, 1000) < ::minRelayTxFee)
                {
                    return InitError(strprintf(_("Invalid amount for -maxtxfee=<amount>: '%s' (must be at least the minrelay fee of %s to prevent stuck transactions)"),
                                               gArgs.GetArg("-maxtxfee", ""), ::minRelayTxFee.ToString()));
                }
            }

22. 获取 `-permitbaremultisig`、`-datacarrier`、`-datacarriersize`等参数。

        fIsBareMultisigStd = gArgs.GetBoolArg("-permitbaremultisig", DEFAULT_PERMIT_BAREMULTISIG);
        fAcceptDatacarrier = gArgs.GetBoolArg("-datacarrier", DEFAULT_ACCEPT_DATACARRIER);
        nMaxDatacarrierBytes = gArgs.GetArg("-datacarriersize", nMaxDatacarrierBytes);

23. 调用 `-SetMockTime` 方法，设置模拟时间。

        SetMockTime(gArgs.GetArg("-mocktime", 0)); 

24. 根据 `-peerbloomfilters` 参数，设置本地支持的服务。

        if (gArgs.GetBoolArg("-peerbloomfilters", DEFAULT_PEERBLOOMFILTERS))
            nLocalServices = ServiceFlags(nLocalServices | NODE_BLOOM);

25. 检测 `-rpcserialversion` 参数是否小于0，是否大于1。

        if (gArgs.GetArg("-rpcserialversion", DEFAULT_RPC_SERIALIZE_VERSION) < 0)
            return InitError("rpcserialversion must be non-negative.");

        if (gArgs.GetArg("-rpcserialversion", DEFAULT_RPC_SERIALIZE_VERSION) > 1)
            return InitError("unknown rpcserialversion requested.");

26. 获取 `-maxtipage` 参数值。

        nMaxTipAge = gArgs.GetArg("-maxtipage", DEFAULT_MAX_TIP_AGE);

27. 处理 `-mempoolreplacement` 参数。

    `-mempoolreplacement` 参数表示是否启用交易池交易替换。

        fEnableReplacement = gArgs.GetBoolArg("-mempoolreplacement", DEFAULT_ENABLE_REPLACEMENT);
        if ((!fEnableReplacement) && gArgs.IsArgSet("-mempoolreplacement")) {
            std::string strReplacementModeList = gArgs.GetArg("-mempoolreplacement", "");  // default is impossible
            std::vector<std::string> vstrReplacementModes;
            boost::split(vstrReplacementModes, strReplacementModeList, boost::is_any_of(","));
            fEnableReplacement = (std::find(vstrReplacementModes.begin(), vstrReplacementModes.end(), "fee") != vstrReplacementModes.end());
        }

28. 处理 `-vbparams` 参数。

    `-vbparams` 参数代表了对于指定的版本位部署，使用给定的开始/结束时间。

    重载版本位只在回归测试模式下才允许，否则会抛出初始异常错误。在回归测试模式下检查所有指定的版本位。

        if (gArgs.IsArgSet("-vbparams")) {
            // Allow overriding version bits parameters for testing
            if (!chainparams.MineBlocksOnDemand()) {
                return InitError("Version bits parameters may only be overridden on regtest.");
            }
            for (const std::string& strDeployment : gArgs.GetArgs("-vbparams")) {
                std::vector<std::string> vDeploymentParams;
                boost::split(vDeploymentParams, strDeployment, boost::is_any_of(":"));
                if (vDeploymentParams.size() != 3) {
                    return InitError("Version bits parameters malformed, expecting deployment:start:end");
                }
                int64_t nStartTime, nTimeout;
                if (!ParseInt64(vDeploymentParams[1], &nStartTime)) {
                    return InitError(strprintf("Invalid nStartTime (%s)", vDeploymentParams[1]));
                }
                if (!ParseInt64(vDeploymentParams[2], &nTimeout)) {
                    return InitError(strprintf("Invalid nTimeout (%s)", vDeploymentParams[2]));
                }
                bool found = false;
                for (int j=0; j<(int)Consensus::MAX_VERSION_BITS_DEPLOYMENTS; ++j)
                {
                    if (vDeploymentParams[0].compare(VersionBitsDeploymentInfo[j].name) == 0) {
                        UpdateVersionBitsParameters(Consensus::DeploymentPos(j), nStartTime, nTimeout);
                        found = true;
                        LogPrintf("Setting version bits activation parameters for %s to start=%ld, timeout=%ld\n", vDeploymentParams[0], nStartTime, nTimeout);
                        break;
                    }
                }
                if (!found) {
                    return InitError(strprintf("Invalid deployment (%s)", vDeploymentParams[0]));
                }
            }
        }

### 第4步，检查相关的加密函数（`src/bitcoind.cpp`）

`AppInitSanityChecks` 函数初始相关的加密曲线与函数，并且确保只有 Bitcoind 在运行。

1.  调用 `SHA256AutoDetect()` 方法，探测使用的 SHA256 算法。

2.  调用 `RandomInit` 方法，初始化随机数。

3.  调用 `ECC_Start` 方法，初始化椭圆曲线。

4.  调用 `globalVerifyHandle.reset(new ECCVerifyHandle())` 方法，重置验证处理器。

5.  调用 `InitSanityCheck` 方法，进行完整性检查。主要是进行各种底层检查。

6.  调用 `LockDataDirectory` 方法，锁定数据目录，确保只有一个 bitcoind 进程在使用数据目录。


### 第4a 步，应用程序初始化（`src/init.cpp::AppInitMain()`）

`AppInitMain` 函数是应用初始化的主体，包括本步骤在内的以下步骤的主体都是在这个函数内部执行。

1.  调用 `Params` 函数，获取 `chainparams`。

    方法定义在 `src/chainparams.cpp` 文件中。这个变量主要是包含一些共识的参数，自身是根据选择不同的网络 `main`、`testnet` 或者 `regtest` 来生成不同的参数。

2.  如果是非 Windows 系统，则调用 `CreatePidFile` 函数，创建进程的PID文件。

    pid 文件简介如下：

    -   pid文件的内容

        pid文件为文本文件，内容只有一行, 记录了该进程的ID。 用cat命令可以看到。

    -   pid文件的作用

        防止进程启动多个副本。只有获得pid文件(固定路径固定文件名)写入权限(F_WRLCK)的进程才能正常启动并把自身的PID写入该文件中。其它同一个程序的多余进程则自动退出。

3.  如果命令行指定了 `shrinkdebugfile` 参数或默认的调试文件，则调用日志对象的 `ShrinkDebugFile` 方法，处理 `debug.log` 文件。

    如果日志长度小于11MB，那么就不做处理；否则读取文件的最后 `RECENT_DEBUG_HISTORY_SIZE` 10M 内容，重新保存到debug.log文件中。

4.  调用日志对象的 `OpenDebugLog` 方法，打开日志文件。如果不能打开则抛出异常。

    3，4 两步代码如下：

        if (g_logger->m_print_to_file) {
            if (gArgs.GetBoolArg("-shrinkdebugfile", g_logger->DefaultShrinkDebugFile())) {
                // Do this first since it both loads a bunch of debug.log into memory,
                // and because this needs to happen before any other debug.log printing
                g_logger->ShrinkDebugFile();
            }
            if (!g_logger->OpenDebugLog()) {
                return InitError(strprintf("Could not open debug log file %s",
                                           g_logger->m_file_path.string()));
            }
        }

5.  调用 `InitSignatureCache` 函数，设置签名缓冲区大小。

    方法内部会根据 `-maxsigcachesize` 参数和默认签名缓冲区的大小来设置最终签名缓冲区大小。

6.  调用 `InitScriptExecutionCache` 函数，设置脚本执行缓存区大小。

    方法内部会根据 `-maxsigcachesize` 参数和默认签名缓冲区的大小来设置最终脚本执行缓冲区大小。

7.  创建指定数量的签名验证线程，并放入线程组。

    具体创建多少个线程，即`nScriptCheckThreads` 变量在前面根据命令行参数 `par` 进行设置。创建线程代码如下：

        if (nScriptCheckThreads) {
            for (int i=0; i<nScriptCheckThreads-1; i++)
                threadGroup.create_thread(&ThreadScriptCheck);
        }

    线程内部调用 `ThreadScriptCheck` 函数进行执行。 `ThreadScriptCheck` 函数过程如下：

    -   首先调用 `RenameThread` 函数（内部调用 `pthread_setname_np` 函数）将当前线程重命名为 `bitcoin-scriptch`。

    -   然后调用 `CCheckQueue` 队列对象的 `Thread` 方法，开启内部循环。

        `Thread` 方法又调用内部私有方法 `Loop` 方法，生成一个脚本验证工作者，然后进行无限循环，在循环内部调用工作者的 `wait(lock)` 方法，从而线程进入阻塞，直到有新的任务被加到队列中中时，才会被唤醒执行任务。

8.  创建一个轻量级的任务定时线程。

    具体代码如下：

        CScheduler::Function serviceLoop = boost::bind(&CScheduler::serviceQueue, &scheduler);
        threadGroup.create_thread(boost::bind(&TraceThread<CScheduler::Function>, "scheduler", serviceLoop));

    代码首先调用 `boost::bind` 方法，生成 `CScheduler` 对象 `serviceQueue` 方法的替代方法。然后调用 `threadGroup.create_thread` 方法，创建一个线程。

    线程执行的方法是 `boost::bind` 返回的替代方法，`bind` 方法的第一个参数为 `TraceThread` 函数，第二个参数为线程的名字，第三个参数为`serviceQueue` 方法的替代方法。

    `TraceThread` 函数内部调用 `RenameThread` 方法修改线程名字，此处线程名字修改为 `bitcoin-scheduler`；然后执行传入的可调用对象，此处为前面的替代方法，即 `CScheduler` 对象 `serviceQueue` 方法。

    `serviceQueue` 方法主体是一个无限循环方法，如果队列为空，则进程进入阻塞，直到队列有任务，则醒来执行任务，并把任务从队列中移除。

    > `bind` 方法简介：

    > bind并不是一个单独的类或函数，而是非常庞大的家族，依据绑定的参数的个数和要绑定的调用对象的类型，总共有数十种不同的形式，编译器会根据具体的绑定代码制动确定要使用的正确的形式。

    > bind接收的第一个参数必须是一个可调用的对象f，包括函数、函数指针、函数对象、和成员函数指针，之后bind最多接受9个参数，参数数量必须与f的参数数量相等，这些参数被传递给f作为入参。 绑定完成后，bind会返回一个函数对象，它内部保存了f的拷贝，具有operator()，返回值类型被自动推导为f的返回类型。在发生调用时这个函数对象将把之前存储的参数转发给f完成调用。

    > bind的真正威力在于它的占位符，它们分别定义为`_1`，`_2`，`_3` 一直到 `_9`,位于一个匿名的名字空间。占位符可以取代 bind 参数的位置，在发生调用时才接受真正的参数。占位符的名字表示它在调用式中的顺序，而在绑定的表达式中没有没有顺序的要求，`_1`不一定必须第一个出现，也不一定只出现一次。

9.  注册后台信号调度器。
    
    代码如下：

        GetMainSignals().RegisterBackgroundSignalScheduler(scheduler);

    **`GetMainSignals` 方法返回类型为 `CMainSignals` 的静态全局变量 g_signals。`CMainSignals` 拥有一个类型为 `MainSignalsInstance` 的智能指针 m_internals。**

    `MainSignalsInstance` 是一个结构体，包含了系统的主要信号和一个调度器，包括：

    -   UpdatedBlockTip

    -   TransactionAddedToMempool

    -   BlockConnected

    -   BlockDisconnected

    -   TransactionRemovedFromMempool

    -   ChainStateFlushed

    -   Broadcast

    -   BlockChecked

    -   NewPoWValidBlock

    **`RegisterBackgroundSignalScheduler` 方法生成智能指针 m_internals 对象。在第6步，网络初始化时会指定各种处理器**。

    简单介绍下信号槽。什么是信号槽？

    -   简单来说，信号槽是观察者模式的一种实现，或者说是一种升华。

    -   一个信号就是一个能够被观察的事件，或者至少是事件已经发生的一种通知；一个槽就是一个观察者，通常就是在被观察的对象发生改变的时候——也可以说是信号发出的时候——被调用的函数；你可以将信号和槽连接起来，形成一种观察者-被观察者的关系；当事件或者状态发生改变的时候，信号就会被发出；同时，信号发出者有义务调用所有注册的对这个事件（信号）感兴趣的函数（槽）。

    -   信号和槽是多对多的关系。一个信号可以连接多个槽，而一个槽也可以监听多个信号。

    -   另外信号可以有附加信息。

    比特币中使用的是signals2 信号槽。signals2基于Boost的另一个库signals，实现了线程安全的观察者模式。在signals2库中，观察者模式被称为信号/插槽(signals and slots)，他是一种函数回调机制，一个信号关联了多个插槽，当信号发出时，所有关联它的插槽都会被调用。

10.  调用 `GetMainSignals().RegisterWithMempoolSignals` 方法，注册内存池信号处理器。

    方法实现如下：

            void CMainSignals::RegisterWithMempoolSignals(CTxMemPool& pool) {
                pool.NotifyEntryRemoved.connect(boost::bind(&CMainSignals::MempoolEntryRemoved, this, _1, _2));
            }

    `pool.NotifyEntryRemoved` 变量定义如下：

        boost::signals2::signal<void (CTransactionRef, MemPoolRemovalReason)> NotifyEntryRemoved;

    上面 `connect` 方法，把插槽连接到信号上，相当于为信号(事件)增加了一个处理器，本例中处理器为 `CMainSignals::MempoolEntryRemoved` 返回的 bind 方法。每当有信号产生时，就会调用这个方法。

11. 调用内联函数 `RegisterAllCoreRPCCommands` ，注册所有核心的 RPC 命令。

    这里及下面的钱包注册 RPC 命令 只讲下每个 RPC 的作用，具体代码与使用后面会进行细讲。如果想要查看系统提供的 RCP 命令/接口，在命令行下输入 `./src/bitcoin-cli -regtest help` 就会显示所有非隐藏的 RPC 命令。如果想要显示某个具体的 RPC 接口，比如 `getblockchaininfo`，执行如下命令 `./src/bitcoin-cli -regtest help getblockchaininfo`，即可显示指定 RPC 的相关信息。

    -   第一步，调用 `RegisterBlockchainRPCCommands` 方法，注册所有关于区块链的 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        区块链相关的 RPC 命令有如下一些：

        -   getblockchaininfo，返回一个包含区块链各种状态信息的对象。

        -   getchaintxstats，计算有关链中交易总数和费率的统计数据。

        -   getbestblockhash，返回最长区块链的最佳高度区块的哈希。

        -   getblockstats，计算给定窗口的区块统计信息。

        -   getblockcount，返回最长区块链的区块数量。

        -   getblock，返回指定区块的数据。

        -   getblockhash，返回区块哈希值。

        -   getblockheader，返回指定区块的头部。

        -   getchaintips，返回所有区块树的顶端区块信息，包括最佳区块链，孤儿区块链等。

        -   getdifficulty，返回POW难度值。

        -   getmempoolancestors

        -   getmempooldescendants

        -   getmempoolentry，返回给定交易的交易池数据。

        -   getmempoolinfo，返回交易池活跃状态的详细信息。

        -   getrawmempool，返回交易池中的所有交易ID。

        -   gettxout，返回未花费交易输出的详细信息。

        -   gettxoutsetinfo，返回未花费交易输出的统计信息。

        -   pruneblockchain，修剪区块链。

        -   savemempool，将内存池转储到磁盘。

        -   verifychain，验证区块链数据库。

        -   preciousblock

        -   scantxoutset，扫描符合某些特定描述的未花费的交易输出集。

        -   invalidateblock，永久性地将块标记为无效，就好像它违反了共识规则一样。

        -   reconsiderblock，删除块及其后代的无效状态，重新考虑它们以便进行激活。这可用于撤消 `invalidateblock` 的效果。

        -   waitfornewblock，等待特定的新区块并返回有关它的有用信息。

        -   waitforblock，等待特定的新区块并返回有关它的有用信息。如果超时或区块不存在，则返回指定的区块。

        -   waitforblockheight，等待（最少多少）区块高度并返回区块链顶端的高度和哈希值。如果超时或区块不存在，则返回指定的区块高度和哈希值。

        -   syncwithvalidationinterfacequeue

    -   第二步，调用 `RegisterNetRPCCommands` 方法，注册所有关于网络相关的 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        网络相关的 RPC 命令有如下一些：

        -   getconnectioncount，返回连接到其他节点的数量。

        -   ping，将 ping 请求发送到其他节点，以测量 ping 的时间。

        -   getpeerinfo，返回每一个连接节点的信息。

        -   addnode，添加、或移除、或连接到一个节点一次（目的为了获取其他节点）。

        -   disconnectnode，立即从某个节点断开。

        -   getaddednodeinfo，返回给定节点，或所有节点的信息。

        -   getnettotals，返回网络传输的一些信息。

        -   getnetworkinfo，返回P2P网络的各种状态信息。

        -   setban，向禁止列表中添加或移除IP地址/子网。

        -   listbanned，显示禁止列表的内容

        -   clearbanned，清空禁止列表。

        -   setnetworkactive，禁止或打开所有 P2P 网络活动。

    -   第三步，调用 `RegisterMiscRPCCommands` 方法，注册所有的杂项 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        杂项相关的 RPC 命令有如下一些：

        -   getmemoryinfo，返回一个包含内存使用信息的对象。

        -   logging，获取或设置日志配置。

        -   validateaddress，验证一个比特币地址是否有效。

        -   createmultisig，创建一个多重签名。

        -   verifymessage，验证一个签名过的消息。

        -   signmessagewithprivkey，用私钥签名一个消息。

        -   setmocktime，设置本地时间，只在回归测试下使用。

        -   echo，简单回显输入参数。此命令用于测试。

        -   echojson，简单回显输入参数。此命令用于测试。

    -   第四步，调用 `RegisterMiningRPCCommands` 方法，注册所有关于挖矿相关的 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        挖矿相关的 RPC 命令有如下一些：

        -   getnetworkhashps，根据最后的 n 个区块数据，估算网络每秒哈希速率。

        -   getmininginfo，返回与挖矿相关的信息。

        -   prioritisetransaction，以更高(或更低)的优先级接受已挖掘的块中的事务。

        -   getblocktemplate，获取区块链模板，聚合挖矿会用到这个方法，详见 BIPs 22, 23, 9 和 145。

        -   submitblock，提交一个新区块到网络上。

        -   submitheader，将给定的十六进制数字解码为区标头部，并将其作为候选区块链顶端区块头部提交（如果有效）。

        -   **generatetoaddress，立即挖掘指定数量的区块，在回归测试中可以快速生成区块**。

        -   estimatefee，0.17 版本中被移除。

        -   estimatesmartfee，估计交易所需的费用。

        -   estimaterawfee，估计交易所需的费用。

    -   第五步，调用 `RegisterRawTransactionRPCCommands` 方法，注册所有关于原始交易的 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        原始交易相关的 RPC 命令有如下一些：

        -   getrawtransaction，返回原始交易。

        -   createrawtransaction，基于输入创建交易，返回交易的16进制。

        -   decoderawtransaction，解码原始交易，返回表示原始交易的JSON对象。

        -   decodescript，解码16进制编码过的脚本。

        -   sendrawtransaction，提交一个原始交易到本地接点和网络。

        -   combinerawtransaction，将多个部分签名的交易合并到一个交易中。合并的交易可以是另一个部分签名的交易或完整签署交易。

        -   signrawtransaction，签名一个原始交易。不建议使用。

        -   signrawtransactionwithkey，用私钥签名一个原始交易。

        -   testmempoolaccept，测试一个原始交易是否能被交易池接受。

        -   decodepsbt，返回一个表示序列化的、base64 编码过的部分签名交易对象。

        -   combinepsbt，合并多个部分签名的交易到一个交易中。

        -   finalizepsbt

        -   createpsbt，创建一个部分签名交易格式的交易。

        -   converttopsbt，转化一个网络序列化的交易到 PSBT。

        -   gettxoutproof，区块链方面的。返回包含在区块中的交易的十六进制编码的证明。

        -   verifytxoutproof，区块链方面的。验证区块中交易的证明。

12. 调用钱包接口的 `RegisterRPC` 方法，注册钱包接口的 RPC 命令。

    实现类为 `wallet/init.cpp` 文件中的 `WalletInit` ，方法内部调用 `RegisterWalletRPCCommands` 进行注册，后者又调用 `wallet/rpcwallet.cpp` 文件中的 `RegisterWalletRPCCommands` 方法，完成注册钱包的 RPC 命令。

    钱包相关的 RPC 命令有如下一些：

    -   fundrawtransaction，添加一个输入到交易中，直到交易可以满足输出。

    -   walletprocesspsbt，用钱包里面的输入来更新交易，然后签名输入。

    -   walletcreatefundedpsbt，以部分签名格式（PSBT）创建和funds交易。

    -   resendwallettransactions，立即广播未确认的交易到所有节点。

    -   abandontransaction，将钱包内的交易标记为已放弃。

    -   abortrescan，停止当前钱包扫描。

    -   addmultisigaddress，添加一个 nrequired-to-sign 多重签名地址到钱包。

    -   addwitnessaddress，不建议使用。

    -   backupwallet，备份钱包。

    -   bumpfee

    -   **createwallet，创建并加载一个新钱包**。

        系统会创建一个默认的钱包，名字为空。可以用 `listwallets` 显示所有加载的钱包。可以用 `importprivkey` 命令添加一个私钥到钱包。

        当有多个钱包时，为了操作某个特定钱包，需要使用 `-rpcwallet=钱包名字`，比如：显示默认钱包的命令为：`./src/bitcoin-cli -regtest  -rpcwallet= getwalletinfo`。

    -   dumpprivkey，显示与地址相关联的私钥，`importprivkey` 可以使用这个输出。

    -   dumpwallet，将所有钱包密钥以人类可读的格式转储到服务器端文件。

    -   encryptwallet，用密码加密钱包。

    -   getaddressinfo，显示比特币地址信息。

    -   getbalance，返回总的可用余额。

    -   **getnewaddress，返回一个新的比特币地址**。

        生成地址的过程会先生成私钥，可以通过 `dumpprivkey` 命令来显示与之相关的私钥，可以通过 `setlabel` 命令设置与给定地址相关的标签。

    -   getrawchangeaddress，返回一个新的比特币地址用于找零。这个用于原始交易，不是常规使用。

    -   getreceivedbyaddress，返回至少具有指定确认的交易中给定地址收到的总金额。

    -   gettransaction，返回钱包中指定交易的详细信息。

    -   getunconfirmedbalance，返回未确认的余额总数。

    -   getwalletinfo，返回钱包的信息。

    -   importmulti，导入地址或脚本，以 one-shot-only 方式重新扫描所有地址。

    -   **importprivkey，添加一个私钥到钱包**。

        对多个钱包来说要在命令行需要使用 `-rpcwallet=钱包名字` 来指定使馆名字。比如：`./src/bitcoin-cli -regtest -rpcwallet= importprivkey "cQM91nga98fMG2xGQHe6LYVH46Yo8tQbHBNQqwMNnrFZPcUs3MMf"` ，在执行这个命令时记得要换成你的私钥。

    -   importwallet，从转储文件中导入钱包。

    -   importaddress，添加一个可以查看的地址或脚本（十六进制），就好像它在钱包里但不能用来支付。

    -   importprunedfunds

    -   importpubkey，添加一个可以查看的公钥，就好像它在钱包里但不能用来支付。

    -   keypoolrefill

    -   listaddressgroupings

    -   listlockunspent，返回未花费输出的列表。

    -   listreceivedbyaddress，列出接收地址的余额。

    -   listsinceblock，获取指定区块以来的所有交易，如果省略区块，则获取所有交易。

    -   listtransactions，返回指定数量的最近交易，跳过指定账户的第一个开始的交易。

    -   listunspent，返回未花费交易输出。

    -   listwallets，返回当前已经的钱包列表。

    -   loadwallet，从钱包文件或目录中加载钱包。

    -   lockunspent，更新暂时不能花费的输出列表。临时解锁或锁定特定的交易输出。

    -   sendmany

    -   sendtoaddress，发送一定的币到指定的地址。

    -   settxfee，设置每 kb 交易费用。

    -   signmessage，用某个地址的私钥签名消息。

    -   signrawtransactionwithwallet，签名原始交易的输入。

    -   unloadwallet，卸载请求端点引用的钱包，否则卸载参数中指定的钱包。

    -   walletlock，从内存中移除钱包的加密，锁定钱包。

    -   walletpassphrasechange，更新钱包的密码。

    -   walletpassphrase，在内存中存储钱包的解密密钥。

    -   removeprunedfunds，从钱包中删除指定的交易。

    -   rescanblockchain，重新扫描本地区块链进行钱包相关交易。

    -   sethdseed，设置或生成确定性分层性钱包的种子。

    -   getaccountaddress，不建议使用，即将移除。

    -   getaccount，不建议使用，即将移除。

    -   getaddressesbyaccount，不建议使用，即将移除。

    -   getreceivedbyaccount，不建议使用，即将移除。

    -   listaccounts，不建议使用，即将移除。

    -   listreceivedbyaccount，不建议使用，即将移除。

    -   setaccount，不建议使用，即将移除。

    -   sendfrom，不建议使用，即将移除。

    -   move，不建议使用，即将移除。

    -   getaddressesbylabel，返回与标签相关的所有地址列表。

    -   getreceivedbylabel，返回与标签相关的、并且至少指定确认的所有交易的比特币数量。

    -   listlabels，返回所有的标签，或与特定用途关联地址相关的标签列表。

    -   listreceivedbylabel，返回与标签对应的接收的交易。

    -   setlabel，设置与给定地址相关的标签。

    -   generate，立即挖出指定的区块（在RPC返回之前）到钱包中指定的地址。

13. 如果命令参数指定 `server` ，则调用 `AppInitServers` 方法，注册服务器。

    具体代码如下：

        if (gArgs.GetBoolArg("-server", false))
        {
            uiInterface.InitMessage_connect(SetRPCWarmupStatus);
            if (!AppInitServers())
                return InitError(_("Unable to start HTTP server. See debug log for details."));
        }

    `AppInitServers` 方法内处理流程如下：

    -   调用 `RPCServer::OnStarted` 方法，设置 RPC 服务器启动时的处理方法。

        具体处理方法如下，以后再讲这个方法：

            static void OnRPCStarted()
            {
                uiInterface.NotifyBlockTip_connect(&RPCNotifyBlockChange);
            }

    -   调用 `RPCServer::OnStopped` 方法，设置 RPC 服务器关闭时的处理方法。

        具体处理方法如下，以后再讲这个方法：

            static void OnRPCStopped()
            {
                uiInterface.NotifyBlockTip_disconnect(&RPCNotifyBlockChange);
                RPCNotifyBlockChange(false, nullptr);
                g_best_block_cv.notify_all();
                LogPrint(BCLog::RPC, "RPC stopped.\n");
            }

    -   调用 `InitHTTPServer` 方法，初始化 HTTP 服务器。

            if (!InitHTTPServer())
                return false;

        `InitHTTPServer` 方法首先会调用 `InitHTTPAllowList` 方法初始化允许 JSON-RPC 调用的地址列表。然后生成一个 HTTP 服务器，并设置服务器的超时时间、最大头部大小、最大消息体大小、绑定到指定的地址上（以便允许这些地址发起请求）。最后，生成 HTTP 工作者队列。

    -   调用 `StartRPC` 方法，启动 RPC 信号监听。

    -   调用 `StartHTTPRPC` 方法，启动 HTTP RPC 服务器。

        具体代码如下：

            if (!StartHTTPRPC())
                return false;

        `StartHTTPRPC` 方法处理如下：

        -   首先，调用 `InitRPCAuthentication` 方法，设置 JSON-RPC 调用的鉴权方法。

        -   然后， `RegisterHTTPHandler` 方法，注册 `/` 请求处理方法为 `HTTPReq_JSONRPC` 方法。

        -   再然后，调用 `RegisterHTTPHandler` 方法，注册 `/wallet/` 请求处理方法为 `HTTPReq_JSONRPC` 方法。

    -   如果命令参数指定 `rest`，调用 `StartREST` 方法，设置 `/rest/xxx` 一系列 HTTP 请求的处理器。

    -   调用 `StartHTTPServer` 方法，启动 HTTP 服务器。

        `StartHTTPServer` 方法代码如下：

            void StartHTTPServer()
            {
                LogPrint(BCLog::HTTP, "Starting HTTP server\n");
                int rpcThreads = std::max((long)gArgs.GetArg("-rpcthreads", DEFAULT_HTTP_THREADS), 1L);
                LogPrintf("HTTP: starting %d worker threads\n", rpcThreads);
                std::packaged_task<bool(event_base*)> task(ThreadHTTP);
                threadResult = task.get_future();
                threadHTTP = std::thread(std::move(task), eventBase);

                for (int i = 0; i < rpcThreads; i++) {
                    g_thread_http_workers.emplace_back(HTTPWorkQueueRun, workQueue);
                }
            }

        下面简单讲述下方法内部的处理：

        -   首先，根据命令参数获取处理 RPC 命令的线程数量。

        -   然后，生成一个任务对象 task，从而得到一个事件分发线程。

        -   最好，生成指定数量的处理 RPC 命令的线程。

### 第5步，验证钱包数据库完整性（`src/init.cpp::AppInitMain()`）

调用钱包接口的 `Verify` 方法，验证钱包数据库。实现类为 `wallet/init.cpp` 文件中的 `WalletInit` ，方法处理流程如下：

-   检查命令行指定了禁止钱包 `disablewallet`，如果禁止，则直接返回。

    代码如下：

        if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
            return true;
        }

-   如果设置了钱包路径 `walletdir`，则检查钱包数据库目录是否存在，是否为目录、且是否为常规的的路径。

    代码如下：

        if (gArgs.IsArgSet("-walletdir")) {
            fs::path wallet_dir = gArgs.GetArg("-walletdir", "");
            if (!fs::exists(wallet_dir)) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" does not exist"), wallet_dir.string()));
            } else if (!fs::is_directory(wallet_dir)) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" is not a directory"), wallet_dir.string()));
            } else if (!wallet_dir.is_absolute()) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" is a relative path"), wallet_dir.string()));
            }
        }

-   检查所有的钱包文件。

    首先，确保钱包不存在名称相同的。然后，调用 `CWallet::Verify` 方法检查钱包的路径。

    代码如下：

        for (const auto& wallet_file : wallet_files) {
            fs::path wallet_path = fs::absolute(wallet_file, GetWalletDir());

            if (!wallet_paths.insert(wallet_path).second) {
                return InitError(strprintf(_("Error loading wallet %s. Duplicate -wallet filename specified."), wallet_file));
            }

            std::string error_string;
            std::string warning_string;
            bool verify_success = CWallet::Verify(wallet_file, salvage_wallet, error_string, warning_string);
            if (!error_string.empty()) InitError(error_string);
            if (!warning_string.empty()) InitWarning(warning_string);
            if (!verify_success) return false;
        }

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

8.  处理洋葱网络。 如果指定了 `onion` 参数，则处理洋葱网络的相关设置。

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

9.  处理通过 `-externalip` 参数设置的外部 IP地址。

    获取并遍历所有指定的外部地址，进行如下处理：调用 `Lookup` 方法进行DNS 查找。如果成功则调用 `AddLocal` 方法，保存新的地址。否则，抛出初始化错误。

        for (const std::string& strAddr : gArgs.GetArgs("-externalip")) {
            CService addrLocal;
            if (Lookup(strAddr.c_str(), addrLocal, GetListenPort(), fNameLookup) && addrLocal.IsValid())
                AddLocal(addrLocal, LOCAL_MANUAL);
            else
                return InitError(ResolveErrMsg("externalip", strAddr));
        }

10.  如果设置了 `maxuploadtarget` 参数，则设置最大出站限制。

        if (gArgs.IsArgSet("-maxuploadtarget")) {
            nMaxOutboundLimit = gArgs.GetArg("-maxuploadtarget", DEFAULT_MAX_UPLOAD_TARGET)*1024*1024;
        }


### 第7步，加载区块链（`src/init.cpp::AppInitMain()`）

首先，计算缓存的大小。包括：区块索引数据库、区块状态数据库、内存中 UTXO 集。代码如下：

    fReindex = gArgs.GetBoolArg("-reindex", false);
    bool fReindexChainState = gArgs.GetBoolArg("-reindex-chainstate", false);

    // cache size calculations
    int64_t nTotalCache = (gArgs.GetArg("-dbcache", nDefaultDbCache) << 20);
    nTotalCache = std::max(nTotalCache, nMinDbCache << 20); // total cache cannot be less than nMinDbCache
    nTotalCache = std::min(nTotalCache, nMaxDbCache << 20); // total cache cannot be greater than nMaxDbcache
    int64_t nBlockTreeDBCache = std::min(nTotalCache / 8, nMaxBlockDBCache << 20);
    nTotalCache -= nBlockTreeDBCache;
    int64_t nTxIndexCache = std::min(nTotalCache / 8, gArgs.GetBoolArg("-txindex", DEFAULT_TXINDEX) ? nMaxTxIndexCache << 20 : 0);
    nTotalCache -= nTxIndexCache;
    int64_t nCoinDBCache = std::min(nTotalCache / 2, (nTotalCache / 4) + (1 << 23)); // use 25%-50% of the remainder for disk cache
    nCoinDBCache = std::min(nCoinDBCache, nMaxCoinsDBCache << 20); // cap total coins db cache
    nTotalCache -= nCoinDBCache;
    nCoinCacheUsage = nTotalCache; // the rest goes to in-memory cache
    int64_t nMempoolSizeMax = gArgs.GetArg("-maxmempool", DEFAULT_MAX_MEMPOOL_SIZE) * 1000000;
    LogPrintf("Cache configuration:\n");
    LogPrintf("* Using %.1fMiB for block index database\n", nBlockTreeDBCache * (1.0 / 1024 / 1024));
    if (gArgs.GetBoolArg("-txindex", DEFAULT_TXINDEX)) {
        LogPrintf("* Using %.1fMiB for transaction index database\n", nTxIndexCache * (1.0 / 1024 / 1024));
    }

上面代码中首先计算总的缓存大小，其用 nTotalCache 表示，通过-dbcache参数设置，然后这个值要取在 nMinDbCache 和 nMaxDbCache 之间。接下来计算 nBlockTreeDBCache 和 nCoinDBCache 以及 nCoinCacheUsage，并且 `nTotalCache = nBlockTreeDBCache +nCoinDBCache + nCoinCacheUsage` 。

然后，只要加载标志为真且没有收到关闭系统的请求，即进行以下 while 循环。

 -   调用 `UnloadBlockIndex` 方法，卸载区块相关的索引。

    因为循环可能执行多次，所以在每次循环开始都要调用本方法来清除相关的一些变量。

    主要设置包括：活跃区块链的栈顶指针为空、pindexBestInvalid 为空、pindexBestHeader 为空、清空交易池、mapBlocksUnlinked（键为缺少交易的区块索引，值为有对应交易的区块索引）集合为空、vinfoBlockFile 集合为空、最后区块文件句柄 nLastBlockFile 为0、脏区块 setDirtyBlockIndex 集合为空、脏区块文件（句柄）setDirtyFileInfo 集合为空、versionbitscache 缓存清空、清空区块索引集合 mapBlockIndex，等等。

-   重置指向活跃 CCoinsView 的全局智能指针变量 pcoinsTip。

-   重置指向 coins 数据库的全局智能指针变量 pcoinsdbview。

-   重置 CCoinsViewErrorCatcher 的智能指针静态变量 pcoinscatcher。

-   重置指向活跃区块树的全局智能指针变量 pblocktree，并生成新的对象。

    这个对象会读写 `/blocks/index/*` 下面的区块文件。

-   如果 `-reset` 参数为真，那么：

    调用 pblocktree 的 `WriteReindexing` 方法，向数据库中写入数据。并且进一步，如果当前处于修剪模式，调用 `CleanupBlockRevFiles` 方法，清除特定区块的数据文件。

        if (fReset) {
            pblocktree->WriteReindexing(true);
            //If we're reindexing in prune mode, wipe away unusable block files and all undo data files
            if (fPruneMode)
                CleanupBlockRevFiles();
        }

    `WriteReindexing` 方法根据参数值是否为真调用不同方法进行写入。如果参数为真，那么调用 `Write` 方法写入数据；否则调用 `Erase` 方法清除数据。

    `CleanupBlockRevFiles` 方法，首先获取区块的目录，然后用迭代器遍历区块目录。如果当前区块目录是一个常规文件，并且文件名字长度为12，且后缀为 `.dat`，那么进行处理：如果当前文件为区块文件，即 `blk?????.dat` 文件，则放入 mapBlockFiles 集合中，否则，即 `rev?????.dat` 文件，调用 `remove` 方法进行删除。

    遍历 mapBlockFiles 集合，如果当前区块文件的名字不等于变量 nContigCounter 的值，即区块文件名字出现了不连续，那么就删除指定的区块文件。看代码可能更清楚：

        int nContigCounter = 0;
        for (const std::pair<const std::string, fs::path>& item : mapBlockFiles) {
            if (atoi(item.first) == nContigCounter) {
                nContigCounter++;
                continue;
            }
            remove(item.second);
        }

-   如果收到结束请求，则退出循环。

-   调用 `LoadBlockIndex` 方法，加载区块索引。

        if (!LoadBlockIndex(chainparams)) {
            strLoadError = _("Error loading block database");
            break;
        }

    `LoadBlockIndex` 方法里，如果没有设置 `fReindex`，即不重建区块索引，那么调用 `LoadBlockIndexDB` 方法，从数据库文件加载区块索引，即从 `/blocks/index/*` 文件中加载区块。如果加载成功，那么设置区块索引集合 `mapBlockIndex` 为空。如果设置了 `fReindex` 变量，因为以后会进行重建索引，所以这里没有必要先加载索引了。

        bool LoadBlockIndex(const CChainParams& chainparams)
        {
            // Load block index from databases
            bool needs_init = fReindex;
            if (!fReindex) {
                bool ret = LoadBlockIndexDB(chainparams);
                if (!ret) return false;
                needs_init = mapBlockIndex.empty();
            }

            if (needs_init) {
                // Everything here is for *new* reindex/DBs. Thus, though
                // LoadBlockIndexDB may have set fReindex if we shut down
                // mid-reindex previously, we don't check fReindex and
                // instead only check it prior to LoadBlockIndexDB to set
                // needs_init.

                LogPrintf("Initializing databases...\n");
            }
            return true;
        }

-   如果区块索引成功加载，则检查是否包含创世区块。

        if (!mapBlockIndex.empty() && !LookupBlockIndex(chainparams.GetConsensus().hashGenesisBlock)) {
            return InitError(_("Incorrect or no genesis block found. Wrong datadir for network?"));
        }

-   如果某些区块被修剪过（即用户手动删除过某些区块），但又没有处于修剪模式，则退出循环。

        if (fHavePruned && !fPruneMode) {
            strLoadError = _("You need to rebuild the database using -reindex to go back to unpruned mode.  This will redownload the entire blockchain");
            break;
        }

-   如果不重建索引，调用 `LoadGenesisBlock` 加载创世区块。如果失败，则退出循环。

        if (!fReindex && !LoadGenesisBlock(chainparams)) {
            strLoadError = _("Error initializing block database");
            break;
        }

    `LoadGenesisBlock` 方法内部调用 `CChainState::LoadGenesisBlock` 方法进行处理。后者处理逻辑如下：

    -   首先，检查区块索引集合 `mapBlockIndex` 集合中是否包含创世区块。如果包含则支持返回。

    -   然后，调用区块链参数的 `GenesisBlock` 方法，返回其保存的创世区块，并调用强制转化为区块对象。

    -   再然后，调用 `SaveBlockToDisk` 方法，把创世区块保存到硬盘上。

    -   再然后，调用 `AddToBlockIndex` 方法，返回或生成并返回一个区块索引对象 CBlockIndex。

    -   最后，调用 `ReceivedBlockTransactions` 方法，更新区块的相关信息到区块索引对象上。同时使用深度优先搜索方法寻找当前链的所有可能的下一个区块。

    下面我们看下 `ReceivedBlockTransactions` 方法：

    -   设置区块索引对象的交易数量为区块的交易集合的大小。

    -   设置区块索引对象的 nChainTx 为0。

        nChainTx 表示从创世区块到当前区块总共有多少个交易，如果这个值为不等于0，那么说明所有父亲都是有效的。

    -   设置区块索引对象的 nFile ，表示区块所在的具体区块文件。

    -   设置区块索引对象的 nDataPos，表示区块在区块文件中偏移的位置。

    -   设置区块索引对象的 nUndoPos，表示区块在 rev?????.dat 文件中偏移的位置。

    -   设置区块索引对象的 nStatus，表示区块完全可用。

    -   调用 `IsWitnessEnabled` 方法，检查是否开启了隔离见证。如果是，则设置区块索引对象的 nStatus，表示区块支持隔离见证。

    -   调用区块索引对象的 `RaiseValidity` 方法，提高此区块索引的有效性级别。

    -   保存区块索引对象到 setDirtyBlockIndex 集合中。

    -   如果是创世区块或当前区块的父区块已经在链上，那么进行下面的处理。

        生成一个指向区块索引对象指针的队列，然后把当前区块索引放入队列尾部。如果队列不为空，则循环队列处理每一个元素。取得队列中的第一个元素，并从队列中删除。设置当前区块索引对象的 nChainTx 为前一个父区块索引对象的 nChainTx 加上当前区块的交易数量（如果是创世区块，那么父区块索引对象的 nChainTx 为0）；设置区块索引对象的序列号；如果活跃区块链的顶端区块不空或当前区块索引对象在活跃区块链顶端区块之后，则把当前区块索引对象加入到 setBlockIndexCandidates 集合中；不断的从孤立的区块集合 mapBlocksUnlinked 中查找，并将当前链的所有可能的下一个区块保存到 setBlockIndexCandidates 中。
        
            void CChainState::ReceivedBlockTransactions(const CBlock& block, CBlockIndex* pindexNew, const CDiskBlockPos& pos, const Consensus::Params& consensusParams)
            {
                pindexNew->nTx = block.vtx.size();
                pindexNew->nChainTx = 0;
                pindexNew->nFile = pos.nFile;
                pindexNew->nDataPos = pos.nPos;
                pindexNew->nUndoPos = 0;
                pindexNew->nStatus |= BLOCK_HAVE_DATA;
                if (IsWitnessEnabled(pindexNew->pprev, consensusParams)) {
                    pindexNew->nStatus |= BLOCK_OPT_WITNESS;
                }
                pindexNew->RaiseValidity(BLOCK_VALID_TRANSACTIONS);
                setDirtyBlockIndex.insert(pindexNew);

                if (pindexNew->pprev == nullptr || pindexNew->pprev->nChainTx) {
                    // If pindexNew is the genesis block or all parents are BLOCK_VALID_TRANSACTIONS.
                    std::deque<CBlockIndex*> queue;
                    queue.push_back(pindexNew);

                    // Recursively process any descendant blocks that now may be eligible to be connected.
                    while (!queue.empty()) {
                        CBlockIndex *pindex = queue.front();
                        queue.pop_front();
                        pindex->nChainTx = (pindex->pprev ? pindex->pprev->nChainTx : 0) + pindex->nTx;
                        {
                            LOCK(cs_nBlockSequenceId);
                            pindex->nSequenceId = nBlockSequenceId++;
                        }
                        if (chainActive.Tip() == nullptr || !setBlockIndexCandidates.value_comp()(pindex, chainActive.Tip())) {
                            setBlockIndexCandidates.insert(pindex);
                        }
                        std::pair<std::multimap<CBlockIndex*, CBlockIndex*>::iterator, std::multimap<CBlockIndex*, CBlockIndex*>::iterator> range = mapBlocksUnlinked.equal_range(pindex);
                        while (range.first != range.second) {
                            std::multimap<CBlockIndex*, CBlockIndex*>::iterator it = range.first;
                            queue.push_back(it->second);
                            range.first++;
                            mapBlocksUnlinked.erase(it);
                        }
                    }
                } else {
                    if (pindexNew->pprev && pindexNew->pprev->IsValid(BLOCK_VALID_TREE)) {
                        mapBlocksUnlinked.insert(std::make_pair(pindexNew->pprev, pindexNew));
                    }
                }
            }

-   生成两个智能指针对象。

        pcoinsdbview.reset(new CCoinsViewDB(nCoinDBCache, false, fReset || fReindexChainState));
        pcoinscatcher.reset(new CCoinsViewErrorCatcher(pcoinsdbview.get()));

    程序执行到这里，要么是已经重新建立所有区块索引；要么已经将所有区块的索引加载到 mapBlockIndex 中了。

-   升级数据库格式。

        if (!pcoinsdbview->Upgrade()) {
            strLoadError = _("Error upgrading chainstate database");
            break;
        }

-   接下来处理分叉引起的数据库不一致。

        if (!ReplayBlocks(chainparams, pcoinsdbview.get())) {
            strLoadError = _("Unable to replay blocks. You will need to rebuild the database using -reindex-chainstate.");
                break;
        }

    `ReplayBlocks` 函数内部调用 `CChainState` 的同名方法进行处理。后者处理逻辑如下：

    -   生成 CCoinsViewCache 的缓存对象 cache 来缓存 CCoinsView。

    -   调用 `CCoinsView` 子类 `CCoinsViewDB` 的 `GetHeadBlocks` 方法，从数据库中加载 Key 为 `H` 的区块头部哈希，并保存在集合中。

    -   如果前一部的区块头部哈希集合为空，直接返回真。如果集合长度不等于2，则抛出异常。

    -   如果区块索引集合 mapBlockIndex 不包含区块头部哈希集合中第1个，则抛出异常。否则，保存在 pindexNew 变量中。

    -   如果区块头部哈希集合中第2个的内容不为0，进行如下的检查：如果这个值不在区块索引集合 mapBlockIndex 中，则抛出异常。否则，保存在 pindexOld 变量中，调用 `LastCommonAncestor` 方法，查到 pindexOld 和 pindexNew 这两个区块的最后一个共同祖先，并保存为 pindexFork。

    -   然后，开始回滚到旧分支。只要 pindexOld 不等于最后一个共同祖先，就进行循环处理。具体逻辑如下：首先，如果当前区块索引对象的高度大于0，即不是创世区块，那么调用 `ReadBlockFromDisk` 方法，从硬盘加载指定的区块，然后调用 `DisconnectBlock` 方法，断开这个区块连接；其次，保存当前区块索引对象的前父对象为当前索引对象。

    -   最后，从最后一个共同祖先向前滚动到区块链的顶端。

    以上处理说起来可能不是太清楚，下面是具体代码，可以参考代码进行理解。

        bool CChainState::ReplayBlocks(const CChainParams& params, CCoinsView* view)
        {
            LOCK(cs_main);

            CCoinsViewCache cache(view);

            std::vector<uint256> hashHeads = view->GetHeadBlocks();
            if (hashHeads.empty()) return true; // We're already in a consistent state.
            if (hashHeads.size() != 2) return error("ReplayBlocks(): unknown inconsistent state");

            uiInterface.ShowProgress(_("Replaying blocks..."), 0, false);
            LogPrintf("Replaying blocks\n");

            const CBlockIndex* pindexOld = nullptr;  // Old tip during the interrupted flush.
            const CBlockIndex* pindexNew;            // New tip during the interrupted flush.
            const CBlockIndex* pindexFork = nullptr; // Latest block common to both the old and the new tip.

            if (mapBlockIndex.count(hashHeads[0]) == 0) {
                return error("ReplayBlocks(): reorganization to unknown block requested");
            }
            pindexNew = mapBlockIndex[hashHeads[0]];

            if (!hashHeads[1].IsNull()) { // The old tip is allowed to be 0, indicating it's the first flush.
                if (mapBlockIndex.count(hashHeads[1]) == 0) {
                    return error("ReplayBlocks(): reorganization from unknown block requested");
                }
                pindexOld = mapBlockIndex[hashHeads[1]];
                pindexFork = LastCommonAncestor(pindexOld, pindexNew);
                assert(pindexFork != nullptr);
            }

            // Rollback along the old branch.
            while (pindexOld != pindexFork) {
                if (pindexOld->nHeight > 0) { // Never disconnect the genesis block.
                    CBlock block;
                    if (!ReadBlockFromDisk(block, pindexOld, params.GetConsensus())) {
                        return error("RollbackBlock(): ReadBlockFromDisk() failed at %d, hash=%s", pindexOld->nHeight, pindexOld->GetBlockHash().ToString());
                    }
                    LogPrintf("Rolling back %s (%i)\n", pindexOld->GetBlockHash().ToString(), pindexOld->nHeight);
                    DisconnectResult res = DisconnectBlock(block, pindexOld, cache);
                    if (res == DISCONNECT_FAILED) {
                        return error("RollbackBlock(): DisconnectBlock failed at %d, hash=%s", pindexOld->nHeight, pindexOld->GetBlockHash().ToString());
                    }
                    // If DISCONNECT_UNCLEAN is returned, it means a non-existing UTXO was deleted, or an existing UTXO was
                    // overwritten. It corresponds to cases where the block-to-be-disconnect never had all its operations
                    // applied to the UTXO set. However, as both writing a UTXO and deleting a UTXO are idempotent operations,
                    // the result is still a version of the UTXO set with the effects of that block undone.
                }
                pindexOld = pindexOld->pprev;
            }

            // Roll forward from the forking point to the new tip.
            int nForkHeight = pindexFork ? pindexFork->nHeight : 0;
            for (int nHeight = nForkHeight + 1; nHeight <= pindexNew->nHeight; ++nHeight) {
                const CBlockIndex* pindex = pindexNew->GetAncestor(nHeight);
                LogPrintf("Rolling forward %s (%i)\n", pindex->GetBlockHash().ToString(), nHeight);
                if (!RollforwardBlock(pindex, cache, params)) return false;
            }

            cache.SetBestBlock(pindexNew->GetBlockHash());
            cache.Flush();
            uiInterface.ShowProgress("", 100, false);
            return true;
        }

-   当系统走到这一步时，硬盘上的 coinsdb 数据库已经片于一致状态了。现在可以创建指向活跃 CCoinsView 的全局智能指针变量 pcoinsTip。

        pcoinsTip.reset(new CCoinsViewCache(pcoinscatcher.get()));

-   如果 coins 视图不空，那么加载指向最佳区块链的栈顶区块，即最高区块。

        bool is_coinsview_empty = fReset || fReindexChainState || pcoinsTip->GetBestBlock().IsNull();
        if (!is_coinsview_empty) {
            // LoadChainTip sets chainActive based on pcoinsTip's best block
            if (!LoadChainTip(chainparams)) {
                strLoadError = _("Error initializing block database");
                break;
            }
            assert(chainActive.Tip() != nullptr);
        }

    在 `LoadChainTip` 方法中，首先检查活跃区块链的顶端区块，如果存在且其哈希等于 pcoinsTip 指向的最佳区块，即最佳区块链/活跃区块链的顶端区块存在，直接返回真。否则，调用活跃 coins 视图 pcoinsTip 的 `GetBestBlock` 方法返回最佳区块，如果最佳区块不为空并且当前区块链只有创世区块（即区块映射集合长度等于1），调用 `ActivateBestChain` 方法，激活最佳区块链。调用 `LookupBlockIndex` 方法，查找最佳区块的索引。如果可以找到，则设置为活跃区块链的顶端指示区块。

    具体代码如下所示：

        bool LoadChainTip(const CChainParams& chainparams)
        {
            AssertLockHeld(cs_main);

            if (chainActive.Tip() && chainActive.Tip()->GetBlockHash() == pcoinsTip->GetBestBlock()) return true;

            if (pcoinsTip->GetBestBlock().IsNull() && mapBlockIndex.size() == 1) {
                // In case we just added the genesis block, connect it now, so
                // that we always have a chainActive.Tip() when we return.
                LogPrintf("%s: Connecting genesis block...\n", __func__);
                CValidationState state;
                if (!ActivateBestChain(state, chainparams)) {
                    LogPrintf("%s: failed to activate chain (%s)\n", __func__, FormatStateMessage(state));
                    return false;
                }
            }

            // Load pointer to end of best chain
            CBlockIndex* pindex = LookupBlockIndex(pcoinsTip->GetBestBlock());
            if (!pindex) {
                return false;
            }
            chainActive.SetTip(pindex);

            g_chainstate.PruneBlockIndexCandidates();

            LogPrintf("Loaded best chain: hashBestChain=%s height=%d date=%s progress=%f\n",
                chainActive.Tip()->GetBlockHash().ToString(), chainActive.Height(),
                FormatISO8601DateTime(chainActive.Tip()->GetBlockTime()),
                GuessVerificationProgress(chainparams.TxData(), chainActive.Tip()));
            return true;
        }

-   接下来处理数据缺失的情况。

        if (!fReset) {
            uiInterface.InitMessage(_("Rewinding blocks..."));
            if (!RewindBlockIndex(chainparams)) {
                strLoadError = _("Unable to rewind the database to a pre-fork state. You will need to redownload the blockchain");
                break;
            }
        }

    `RewindBlockIndex` 方法中，会断开区块连接，并从区块索引对象中删除相关的区块索引。

-   最后，进行区块数据验证。具体处理可以看代码，此处不细说。


### 第8步，开始索引（`src/init.cpp::AppInitMain()`）

如果指定了 `-txindex` 参数，则生成交易索引对象 g_txindex，类型为 `TxIndex`；然后调用其 `Start` 方法，开始建立索引。

    if (gArgs.GetBoolArg("-txindex", DEFAULT_TXINDEX)) {
        g_txindex = MakeUnique<TxIndex>(nTxIndexCache, false, fReindex);
        g_txindex->Start();
    }

`start` 方法处理如下：

1.  首先，调用 `RegisterValidationInterface` 方法注册 `TxIndex` 为 `MainSignalsInstance` 上各种事件的信号处理器，在发送信号时会调用这些处理器。

        RegisterValidationInterface(this);

2.  然后，调用 `Init` 方法升级交易索引从老的数据库到新的数据库。

    `TxIndex` 子类重载了这个方法，会调用 `m_db->MigrateData(*pblocktree, chainActive.GetLocator())` 方法来升级数据库。

    然后，调用父类 `BaseIndex` 的同名方法进行处理。在父类的 `Init` 方法中，首先会调用 `ReadBestBlock` 方法从数据库中读取 Key 为 `B` 的区块做为定位器（可能是所有没有分叉的区块）。然后，调用 `FindForkInGlobalIndex` 方法，找到活跃区块链上的分叉前的最后一区块索引（从这个区块产生了分叉）。如果这个索引对应的区块和活跃区块链的顶端区块是相同的，设置同步完成标志为真。

3.  启动一个线程，线程执行的真正方法为 `BaseIndex::ThreadSync`。线程的主要作用在于当没有同步完成时，通过读取活跃区块链的下一个区块来进行同步，并把没有分叉的区块以 Key 为 `B` 写入数据库中。


### 第9步，加载钱包（`src/init.cpp::AppInitMain()`）

调用钱包接口对象的 `Open` 方法，开始加载钱包。具体方法在 `wallet/init.cpp` 文件中。内容如下：

    bool WalletInit::Open() const
    {
        if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
            LogPrintf("Wallet disabled!\n");
            return true;
        }

        for (const std::string& walletFile : gArgs.GetArgs("-wallet")) {
            std::shared_ptr<CWallet> pwallet = CWallet::CreateWalletFromFile(walletFile, fs::absolute(walletFile, GetWalletDir()));
            if (!pwallet) {
                return false;
            }
            AddWallet(pwallet);
        }

        return true;
    }

首先检查是否禁止钱包，如果禁止直接返回。否则遍历所有钱包，调用 `CWallet::CreateWalletFromFile` 方法，要根据钱包文件生成钱包对象，如果成功生成钱包，则调用 `AddWallet` 方法把钱包加入 `vpwallets` 集合中。


### 第10，数据目录维护（`src/init.cpp::AppInitMain()`）

如果当前为修剪模式，本地服务去掉 `NODE_NETWORK` 标志，然后如果不需要索引则调用 `PruneAndFlush` 函数，修剪并刷新到硬盘中。

    if (fPruneMode) {
        LogPrintf("Unsetting NODE_NETWORK on prune mode\n");
        nLocalServices = ServiceFlags(nLocalServices & ~NODE_NETWORK);
        if (!fReindex) {
            uiInterface.InitMessage(_("Pruning blockstore..."));
            PruneAndFlush();
        }
    }

### 第11步，导入区块（`src/init.cpp::AppInitMain()`）

1.  调用 `CheckDiskSpace` 函数，检查硬盘空间是否足够。

    如果没有足够的硬盘空间，则退出。

2.  检查最佳区块链顶端指示指针是否为空。

    如果顶端打针为空，UI界面进行通知。如果不空，则设置有创世区块，即 `fHaveGenesis` 设为真。

        if (chainActive.Tip() == nullptr) {
            uiInterface.NotifyBlockTip_connect(BlockNotifyGenesisWait);
        } else {
            fHaveGenesis = true;
        }
    
3.  如果指定了 `blocknotify` 参数，设置界面通知为 `BlockNotifyCallback`。

4.  遍历参数 `loadblock` 指定要加载的区块文件，放进向量变量 `vImportFiles` 集合中。然后调用 `threadGroup.create_thread` 方法，创建一个线程。线程执行的函数为 `ThreadImport`，参数为要加载的区块文件。

        std::vector<fs::path> vImportFiles;
        for (const std::string& strFile : gArgs.GetArgs("-loadblock")) {
            vImportFiles.push_back(strFile);
        }

        threadGroup.create_thread(boost::bind(&ThreadImport, vImportFiles));

5.  获取 `cs_GenesisWait` 锁，等待创世区块被处理完成。

        {
            WaitableLock lock(cs_GenesisWait);
            // We previously could hang here if StartShutdown() is called prior to
            // ThreadImport getting started, so instead we just wait on a timer to
            // check ShutdownRequested() regularly.
            while (!fHaveGenesis && !ShutdownRequested()) {
                condvar_GenesisWait.wait_for(lock, std::chrono::milliseconds(500));
            }
            uiInterface.NotifyBlockTip_disconnect(BlockNotifyGenesisWait);
        }


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

### 第13步，结束启动（`src/init.cpp::AppInitMain()`）

1.  调用 `SetRPCWarmupFinished()` 方法，设置热身结束。

    方法内部主要设置 `fRPCInWarmup` 变量为假，表示热身结束。

2.  调用钱包接口对象的 `Start` 方法，开始进行钱包相关的处理，并定时刷新钱包数据到数据库中。

    代码如下：

        for (const std::shared_ptr<CWallet>& pwallet : GetWallets()) {
            pwallet->postInitProcess();
        }

        // Run a thread to flush wallet periodically
        scheduler.scheduleEvery(MaybeCompactWalletDB, 500);

    方法中，首先便利所有钱包对象，调用其 `postInitProcess` 方法，进行后初始化设置。主要是把钱包中存在，但是交易池中不存在的交易添加到交易池中。

    然后，设置调度器定时调用 `MaybeCompactWalletDB` 方法，刷新钱包数据到数据库中。

