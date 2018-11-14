#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


## 第4步，检查相关的加密函数（`src/bitcoind.cpp`）

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

    **`GetMainSignals` 方法返回类型为 `CMainSignals` 的静态全局变量（系统启动时预先进行初始化） g_signals。`CMainSignals` 拥有一个类型为 `MainSignalsInstance` 的智能指针 m_internals。**

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

        -   getblockchaininfo

            返回一个包含区块链各种状态信息的对象。

        -   getchaintxstats

            计算有关链中交易总数和费率的统计数据。

        -   getbestblockhash

            返回最长区块链的最佳高度区块的哈希。

        -   getblockstats

            计算给定窗口的区块统计信息。

        -   getblockcount

            返回最长区块链的区块数量。

        -   getblock

            返回指定区块的数据。

        -   getblockhash

            返回区块哈希值。

        -   getblockheader

            返回指定区块的头部。

        -   getchaintips

            返回所有区块树的顶端区块信息，包括最佳区块链，孤儿区块链等。

        -   getdifficulty

            返回POW难度值。

        -   getmempoolancestors

        -   getmempooldescendants

        -   getmempoolentry

            返回给定交易的交易池数据。

        -   getmempoolinfo

            返回交易池活跃状态的详细信息。

        -   getrawmempool

            返回交易池中的所有交易ID。

        -   gettxout

            返回未花费交易输出的详细信息。

        -   gettxoutsetinfo

            返回未花费交易输出的统计信息。

        -   pruneblockchain

            修剪区块链。

        -   savemempool

            将内存池转储到磁盘。

        -   verifychain

            验证区块链数据库。

        -   preciousblock

        -   scantxoutset

            扫描符合某些特定描述的未花费的交易输出集。

        -   invalidateblock

            永久性地将块标记为无效，就好像它违反了共识规则一样。

        -   reconsiderblock

            删除块及其后代的无效状态，重新考虑它们以便进行激活。这可用于撤消 `invalidateblock` 的效果。

        -   waitfornewblock

            等待特定的新区块并返回有关它的有用信息。

        -   waitforblock

            等待特定的新区块并返回有关它的有用信息。如果超时或区块不存在，则返回指定的区块。

        -   waitforblockheight

            等待（最少多少）区块高度并返回区块链顶端的高度和哈希值。如果超时或区块不存在，则返回指定的区块高度和哈希值。

        -   syncwithvalidationinterfacequeue

    -   第二步，调用 `RegisterNetRPCCommands` 方法，注册所有关于网络相关的 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        网络相关的 RPC 命令有如下一些：

        -   getconnectioncount

            返回连接到其他节点的数量。

        -   ping

            将 ping 请求发送到其他节点，以测量 ping 的时间。

        -   getpeerinfo

            返回每一个连接节点的信息。

        -   addnode

            添加、或移除、或连接到一个节点一次（目的为了获取其他节点）。

        -   disconnectnode

            立即从某个节点断开。

        -   getaddednodeinfo

            返回给定节点，或所有节点的信息。

        -   getnettotals

            返回网络传输的一些信息。

        -   getnetworkinfo

            返回P2P网络的各种状态信息。

        -   setban

            向禁止列表中添加或移除IP地址/子网。

        -   listbanned

            显示禁止列表的内容

        -   clearbanned

            清空禁止列表。

        -   setnetworkactive

            禁止或打开所有 P2P 网络活动。

    -   第三步，调用 `RegisterMiscRPCCommands` 方法，注册所有的杂项 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        杂项相关的 RPC 命令有如下一些：

        -   getmemoryinfo

            返回一个包含内存使用信息的对象。

        -   logging

            获取或设置日志配置。

        -   validateaddress

            验证一个比特币地址是否有效。

        -   createmultisig

            创建一个多重签名。

        -   verifymessage

            验证一个签名过的消息。

        -   signmessagewithprivkey

            用私钥签名一个消息。

        -   setmocktime

            设置本地时间，只在回归测试下使用。

        -   echo

            简单回显输入参数。此命令用于测试。

        -   echojson

            简单回显输入参数。此命令用于测试。

    -   第四步，调用 `RegisterMiningRPCCommands` 方法，注册所有关于挖矿相关的 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        挖矿相关的 RPC 命令有如下一些：

        -   getnetworkhashps

            根据最后的 n 个区块数据，估算网络每秒哈希速率。

        -   getmininginfo

            返回与挖矿相关的信息。

        -   prioritisetransaction

            以更高(或更低)的优先级接受已挖掘的块中的事务。

        -   getblocktemplate

            获取区块链模板，聚合挖矿会用到这个方法，详见 BIPs 22, 23, 9 和 145。

        -   submitblock

            提交一个新区块到网络上。

        -   submitheader

            将给定的十六进制数字解码为区标头部，并将其作为候选区块链顶端区块头部提交（如果有效）。

        -   generatetoaddress

            **立即挖掘指定数量的区块，在回归测试中可以快速生成区块**。

        -   estimatefee

            0.17 版本中被移除。

        -   estimatesmartfee

            估计交易所需的费用。

        -   estimaterawfee

            估计交易所需的费用。

    -   第五步，调用 `RegisterRawTransactionRPCCommands` 方法，注册所有关于原始交易的 RPC 命令。

        方法内部会遍历 `commands` 数组，把每一个命令保存到 `CRPCTable` 对象的 `mapCommands` 集合中。

        原始交易相关的 RPC 命令有如下一些：

        -   getrawtransaction

            返回原始交易。

        -   createrawtransaction

            基于输入创建交易，返回交易的16进制。

        -   decoderawtransaction

            解码原始交易，返回表示原始交易的JSON对象。

        -   decodescript

            解码16进制编码过的脚本。

        -   sendrawtransaction

            提交一个原始交易到本地接点和网络。

        -   combinerawtransaction

            将多个部分签名的交易合并到一个交易中。合并的交易可以是另一个部分签名的交易或完整签署交易。

        -   signrawtransaction

            签名一个原始交易。不建议使用。

        -   signrawtransactionwithkey

            用私钥签名一个原始交易。

        -   testmempoolaccept

            测试一个原始交易是否能被交易池接受。

        -   decodepsbt

            返回一个表示序列化的、base64 编码过的部分签名交易对象。

        -   combinepsbt

            合并多个部分签名的交易到一个交易中。

        -   finalizepsbt

        -   createpsbt

            创建一个部分签名交易格式的交易。

        -   converttopsbt

            转化一个网络序列化的交易到 PSBT。

        -   gettxoutproof

            区块链方面的。返回包含在区块中的交易的十六进制编码的证明。

        -   verifytxoutproof

            区块链方面的。验证区块中交易的证明。

12. 调用钱包接口的 `RegisterRPC` 方法，注册钱包接口的 RPC 命令。

    实现类为 `wallet/init.cpp` 文件中的 `WalletInit` ，方法内部调用 `RegisterWalletRPCCommands` 进行注册，后者又调用 `wallet/rpcwallet.cpp` 文件中的 `RegisterWalletRPCCommands` 方法，完成注册钱包的 RPC 命令。

    钱包相关的 RPC 命令有如下一些：

    -   fundrawtransaction

        添加一个输入到交易中，直到交易可以满足输出。

    -   walletprocesspsbt

        用钱包里面的输入来更新交易，然后签名输入。

    -   walletcreatefundedpsbt

        以部分签名格式（PSBT）创建和funds交易。

    -   resendwallettransactions

        立即广播未确认的交易到所有节点。

    -   abandontransaction

        将钱包内的交易标记为已放弃。

    -   abortrescan

        停止当前钱包扫描。

    -   addmultisigaddress

        添加一个 nrequired-to-sign 多重签名地址到钱包。

    -   addwitnessaddress

        不建议使用。

    -   backupwallet

        备份钱包。

    -   bumpfee

    -   createwallet

        **创建并加载一个新钱包**。 系统会自动创建一个默认的钱包，名字为空。

            ./src/bitcoin-cli -regtest  createwallet test

       可以用 `listwallets` 显示所有加载的钱包，可以用 `importprivkey` 命令添加一个私钥到钱包。

        当有多个钱包时，为了操作某个特定钱包，需要使用 `-rpcwallet=钱包名字`，比如：

            ./src/bitcoin-cli -regtest -rpcwallet= getwalletinfo
            ./src/bitcoin-cli -regtest -rpcwallet=test getwalletinfo

    -   dumpprivkey

        显示与地址相关联的私钥，`importprivkey` 可以使用这个输出。

    -   dumpwallet

        将所有钱包密钥以人类可读的格式转储到服务器端文件。

    -   encryptwallet

        用密码加密钱包。

    -   getaddressinfo

        显示比特币地址信息。

    -   getbalance

        返回总的可用余额。

    -   getnewaddress

        **返回一个新的比特币地址**。

            ./src/bitcoin-cli -regtest -rpcwallet=test getnewaddress

        `-rpcwallet` 参数一定要放在 RPC 命令之前。
        
        生成地址的过程会先生成私钥，可以通过 `dumpprivkey` 命令来显示与之相关的私钥，可以通过 `setlabel` 命令设置与给定地址相关的标签。

    -   getrawchangeaddress

        返回一个新的比特币地址用于找零。这个用于原始交易，不是常规使用。

    -   getreceivedbyaddress

        返回至少具有指定确认的交易中给定地址收到的总金额。

    -   gettransaction

        返回钱包中指定交易的详细信息。

    -   getunconfirmedbalance

        返回未确认的余额总数。

    -   getwalletinfo

        返回钱包的信息。

    -   importmulti

        导入地址或脚本，以 one-shot-only 方式重新扫描所有地址。

    -   importprivkey

        **添加一个私钥到钱包**。

        对多个钱包来说要在命令行需要使用 `-rpcwallet=钱包名字` 来指定使馆名字。比如：`./src/bitcoin-cli -regtest -rpcwallet= importprivkey "cQM91nga98fMG2xGQHe6LYVH46Yo8tQbHBNQqwMNnrFZPcUs3MMf"` ，在执行这个命令时记得要换成你的私钥。

    -   importwallet

        从转储文件中导入钱包。

    -   importaddress

        添加一个可以查看的地址或脚本（十六进制），就好像它在钱包里但不能用来支付。

    -   importprunedfunds

    -   importpubkey

        添加一个可以查看的公钥，就好像它在钱包里但不能用来支付。

    -   keypoolrefill

    -   listaddressgroupings

    -   listlockunspent

        返回未花费输出的列表。

    -   listreceivedbyaddress

        列出接收地址的余额。

    -   listsinceblock

        获取指定区块以来的所有交易，如果省略区块，则获取所有交易。

    -   listtransactions

        返回指定数量的最近交易，跳过指定账户的第一个开始的交易。

    -   listunspent

        返回未花费交易输出。

    -   listwallets

        返回当前已经的钱包列表。

    -   loadwallet

        从钱包文件或目录中加载钱包。

    -   lockunspent

        更新暂时不能花费的输出列表。临时解锁或锁定特定的交易输出。

    -   sendmany

    -   sendtoaddress

        **发送一定的币到指定的地址，即开始进行交易及探矿等**。

    -   settxfee

        设置每 kb 交易费用。

    -   signmessage

        用某个地址的私钥签名消息。

    -   signrawtransactionwithwallet

        签名原始交易的输入。

    -   unloadwallet

        卸载请求端点引用的钱包，否则卸载参数中指定的钱包。

    -   walletlock

        从内存中移除钱包的加密，锁定钱包。

    -   walletpassphrasechange

        更新钱包的密码。

    -   walletpassphrase

        在内存中存储钱包的解密密钥。

    -   removeprunedfunds

        从钱包中删除指定的交易。

    -   rescanblockchain

        重新扫描本地区块链进行钱包相关交易。

    -   sethdseed

        设置或生成确定性分层性钱包的种子。

    -   getaccountaddress

        不建议使用，即将移除。

    -   getaccount

        不建议使用，即将移除。

    -   getaddressesbyaccount

        不建议使用，即将移除。

    -   getreceivedbyaccount

        不建议使用，即将移除。

    -   listaccounts

        不建议使用，即将移除。

    -   listreceivedbyaccount

        不建议使用，即将移除。

    -   setaccount

        不建议使用，即将移除。

    -   sendfrom

        不建议使用，即将移除。

    -   move

        不建议使用，即将移除。

    -   getaddressesbylabel

        返回与标签相关的所有地址列表。

    -   getreceivedbylabel

        返回与标签相关的、并且至少指定确认的所有交易的比特币数量。

    -   listlabels

        返回所有的标签，或与特定用途关联地址相关的标签列表。

    -   listreceivedbylabel

        返回与标签对应的接收的交易。

    -   setlabel

        设置与给定地址相关的标签。

    -   generate

        立即挖出指定的区块（在RPC返回之前）到钱包中指定的地址。

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
