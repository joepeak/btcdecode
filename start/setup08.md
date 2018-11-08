#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第8步，建立索引（`src/init.cpp::AppInitMain()`）

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

