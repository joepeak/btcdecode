#   区块验证相关


##  1、CMainSignals

主信号类，包括了一个指向结构体 `MainSignalsInstance` 的智能指针 `m_internals`，几个私有的友元方法，几个公开的方法。

这个类有一个定义在 `validationInterface.cpp` 文件中的静态全局变量 `g_signals` 在系统启动时预先进行初始化，其内部指向主信号实例 `MainSignalsInstance` 的 `m_internals` 属性在系统启动的第 4a 步中被调用 `RegisterBackgroundSignalScheduler` 方法进行初始化。


私有友元方法

1.  RegisterValidationInterface

2.  UnregisterValidationInterface

3.  UnregisterAllValidationInterfaces

4.  CallFunctionInValidationInterfaceQueue

私有方法

1.  MempoolEntryRemoved

公开方法

1.  RegisterBackgroundSignalScheduler

    初始化类型为 `MainSignalsInstance` 结构体的内部变量 `m_internals`。

    在系统启动的第 4a 步中被调用，具体请参考第 4a 步。

2.  UnregisterBackgroundSignalScheduler

    重置变量 `m_internals`。

3.  FlushBackgroundCallbacks

4.  CallbacksPending

5.  RegisterWithMempoolSignals

    在系统启动的第 4a 步中被调用，具体请参考第 4a 步。

6.  UnregisterWithMempoolSignals

7.  UpdatedBlockTip

8.  TransactionAddedToMempool

9.  BlockConnected

10. BlockDisconnected

11. ChainStateFlushed

12. Broadcast

13. BlockChecked

14. NewPoWValidBlock


##  3、MainSignalsInstance

结构体，定义于 `validationinterface.cpp` 文件中。主体是一些 `boost::signals2::signal` 定义的信号，具体有以下几个：

1.  UpdatedBlockTip

2.  TransactionAddedToMempool

3.  BlockConnected

4.  BlockDisconnected

5.  TransactionRemovedFromMempool

6.  ChainStateFlushed

7.  Broadcast

8.  BlockChecked

9.  NewPoWValidBlock

同时，还有一个指向 `SingleThreadedSchedulerClient` 的内部变量 `m_schedulerClient`，在初始化时被设置。


##  3、CValidationInterface

实现这个类以便订阅在验证过程中产生的各种事件。

事件方法如下：

1.  UpdatedBlockTip

    虚函数，通知监听器区块链顶端向前了。

2.  TransactionAddedToMempool

    虚函数，通知监听器交易被加入到交易池中。

3.  TransactionRemovedFromMempool

    虚函数，通知监听器交易被从交易池移出中。

4.  BlockConnected

    虚函数，通知监听器区块被连接。

5.  BlockDisconnected

    虚函数，通知监听器区块被去掉连接。

6.  ChainStateFlushed

    虚函数，Notifies listeners of the new active block chain on-disk.

7.  ResendWalletTransactions

    虚函数，Tells listeners to broadcast their data.

8.  BlockChecked

    虚函数，Notifies listeners of a block validation result.

9.  NewPoWValidBlock

    虚函数，Notifies listeners that a block which builds directly on our current tip has been received and connected to the headers tree, though not validated yet

友元方法如下：

1.  RegisterValidationInterface

    注册一个验证接口/监听器。

2.  UnregisterValidationInterface

    删除一个验证接口/监听器。

3.  UnregisterAllValidationInterfaces

    删除所有验证接口/监听器。


当前的子类有：`CWallet`、`CZMQNotificationInterface`、`PeerLogicValidation`、`submitblock_StateCatcher` 等。

1.  `CWallet` 实现了以下几个方法：

    -   TransactionAddedToMempool

    -   BlockConnected

    -   BlockDisconnected

    -   TransactionRemovedFromMempool

    -   ResendWalletTransactions


2.  `CZMQNotificationInterface` 实现了以下几个方法：

    -   TransactionAddedToMempool

    -   BlockConnected

    -   BlockDisconnected

    -   UpdatedBlockTip

3.  `PeerLogicValidation` 实现了以下几个方法：

    -   BlockConnected

    -   UpdatedBlockTip

    -   BlockChecked

    -   NewPoWValidBlock

4.  `submitblock_StateCatcher` 实现了以下几个方法：

    -   BlockChecked
