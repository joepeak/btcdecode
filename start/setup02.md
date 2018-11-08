#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


## 第2步，应用初始参数交互设置（`src/bitcoind.cpp`）

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

