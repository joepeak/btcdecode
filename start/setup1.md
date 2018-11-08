#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


## 第1步，应用初始化基本设置（`src/bitcoind.cpp`）

`AppInitBasicSetup` 函数进行基本的设置。

1.  调用 `SetupNetworking` 函数，进行网络设置。

    主要是针对 Win32 系统处理套接字，别的系统直接返回真。

2.  如果不是 WIN32 系统，进行下面的处理：

    -   如果设置 `sysperms` 参数为真，调用 `umask` 函数，设置位码为 077。

    -   调用 `registerSignalHandler` 函数，设置 `SIGTERM` 信号处理器为 `HandleSIGTERM`；`SIGINT` 为 `HandleSIGTERM`；`SIGHUP` 为 `HandleSIGHUP`。
