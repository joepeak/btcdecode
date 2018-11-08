#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


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
