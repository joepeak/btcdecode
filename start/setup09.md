#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第9步，加载钱包（`src/init.cpp::AppInitMain()`）

调用钱包接口对象的 `Open` 方法，开始加载钱包。实现类为 `wallet/init.cpp` 文件中的 `Open` ，方法处理流程如下：


1.  检查启动参数是否禁止钱包 `-disablewallet`。如果是，则直接返回。

        if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
            return true;
        }

2.  `for` 循环用户提供的所有的钱包，此处至少有一个默认钱包，所以肯定会至少循环一次。

    如果用户没有指定启动参数 `-wallet`，则在第三步的第21小步中，调用钱包初始接口对象的 `ParameterInteraction` 方法时，设置启动参数 `-wallet`默认为空字符串，从而在本步时至少创建一个钱包名称为空的默认钱包。

    -   调用 `CreateWalletFromFile` 方法，创建钱包。

            std::shared_ptr<CWallet> pwallet = CWallet::CreateWalletFromFile(walletFile, fs::absolute(walletFile, GetWalletDir()));

    -   如果创建钱包不成功，则返回假。

            if (!pwallet) {
                return false;
            }

    -   调用 `AddWallet` 方法，把生成的钱包加入钱包集合 `vpwallets` 中。

3.  返回真。


`CreateWalletFromFile` 方法的具体讲解会在密钥、地址和钱包部分进行详细的分析，此处略过不讲。

通过本步骤，系统建立了我们的第一个默认的钱包。
