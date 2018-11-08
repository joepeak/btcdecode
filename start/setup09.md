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

首先检查是否禁止钱包，如果禁止直接返回。否则遍历所有钱包，调用 `CWallet::CreateWalletFromFile` 方法，要根据钱包文件生成钱包对象，如果成功生成钱包，则调用 `AddWallet` 方法把钱包加入 `vpwallets` 集合中。

