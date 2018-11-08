#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


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