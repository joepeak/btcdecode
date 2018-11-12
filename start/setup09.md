#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第9步，加载钱包（`src/init.cpp::AppInitMain()`）

调用钱包接口对象的 `Open` 方法，开始加载钱包。具体方法在 `wallet/init.cpp` 文件中。内容如下：

    bool WalletInit::Open() const
    {
        if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
            LogPrintf("Wallet disabled!\n");
            return true;
        }

        for (const std::string& walletFile : gArgs.GetArgs("-wallet")) {
            std::shared_ptr<CWallet> pwallet = CWallet::CreateWalletFromFile(walletFile, fs::absolute(walletFile, GetWalletDir()));
            if (!pwallet) {
                return false;
            }
            AddWallet(pwallet);
        }

        return true;
    }

首先检查是否禁止钱包，如果禁止直接返回。否则遍历启动参数 `-wallet` 指定的所有钱包，调用 `CWallet::CreateWalletFromFile` 方法，要根据钱包文件生成钱包对象。如果成功生成钱包，则调用 `AddWallet` 方法把钱包加入 `vpwallets` 集合中。

如果用户没有指定启动参数 `-wallet`，则在第三步的第21小步中，调用钱包初始接口对象的 `ParameterInteraction` 方法时，设置启动参数 `-wallet`默认为空字符串，从而在本步时至少创建一个钱包名称为空的默认钱包。

`CreateWalletFromFile` 方法的具体讲解会在密钥、地址和钱包部分进行详细的分析，此处略过不讲。

通过本步骤，系统建立了我们的第一个默认的钱包。
