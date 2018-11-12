#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


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
            while (!fHaveGenesis && !ShutdownRequested()) {
                condvar_GenesisWait.wait_for(lock, std::chrono::milliseconds(500));
            }
            uiInterface.NotifyBlockTip_disconnect(BlockNotifyGenesisWait);
        }

