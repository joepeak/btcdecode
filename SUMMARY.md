# 比特币源码情景分析

* [简介](README.md)
    * [序](introduction.md)

* [从零编译比特币](build.md)

* 系统启动流程
    * [系统启动过程之鸟瞰](start/start.md)
    * [第一步：应用基本设置](start/setup01.md)
    * [第二步：初始参数设置](start/setup02.md)
    * [第三步：参数内部转换](start/setup03.md)
    * [第四步：应用程序初始化](start/setup04.md)
    * [第五步：钱包完整性验证](start/setup05.md)
    * [第六步：网络初始化](start/setup06.md)
    * [第七步：加载区块链](start/setup07.md)
    * [第八步：开始建立索引](start/setup08.md)
    * [第九步：加载钱包](start/setup09.md)
    * [第十步：数据目录维护](start/setup10.md)
    * [第十一步：导入区块](start/setup11.md)
    * [第十二步：启动节点](start/setup12.md)
    * [第十三步：启动结束](start/setup13.md)

* P2P 网络建立流程
    * [套接字的处理](net/socket.md)
    * [DNS 种子节点的处理](net/dnsseed.md)
    * [连接到对等节点的处理](net/connode.md)
    * [消息处理上篇-消息线程分析](net/message1.md)
    * [消息处理中篇-协义分析一](net/message2.md)
    * [消息处理下篇-协义分析二](net/message3.md)

* 比特币常用数据结构
	* [网络地址相关](keystruct/addr.md)
	* [对等节点结构](keystruct/cnode.md)
	* [对等节点状态](keystruct/cnodestate.md)
    * [区块索引对象](keystruct/cblockindex.md)
    * [区块对象](keystruct/cblock.md)
    