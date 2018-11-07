# 比特币源码情景分析

* [简介](README.md)
    * [序](introduction.md)


* [从零编译比特币](build.md)

* 系统启动流程
    * [系统启动之鸟瞰](start/start.md)
    * [系统启动之详述](start/setups.md)

* P2P 网络建立流程
    * [套接字的处理](net/socket.md)
    * [DNS 种子节点的处理](net/dnsseed.md)
    * [连接到对等节点的处理](net/connode.md)
    * [消息处理上篇-消息线程分析](net/message1.md)
    * [消息处理中篇-协义分析一](net/message2.md)
    * [消息处理下篇-协义分析之二](net/message3.md)

* 比特币常用数据结构
	* [地址相关结构](keystruct/addr.md)
	* [对等节点结构](keystruct/cnode.md)
	* [对等节点状态](keystruct/cnodestate.md)
    * [区块索引对象](keystruct/cblockindex.md)
    * [区块对象](keystruct/cblock.md)
    