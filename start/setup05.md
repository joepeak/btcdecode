#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第5步，验证钱包数据库完整性（`src/init.cpp::AppInitMain()`）

调用钱包接口的 `Verify` 方法，验证钱包数据库。实现类为 `wallet/init.cpp` 文件中的 `WalletInit` ，方法处理流程如下：

-   检查命令行指定了禁止钱包 `disablewallet`，如果禁止，则直接返回。

    代码如下：

        if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
            return true;
        }

-   如果设置了钱包路径 `walletdir`，则检查钱包数据库目录是否存在，是否为目录、且是否为常规的的路径。

    代码如下：

        if (gArgs.IsArgSet("-walletdir")) {
            fs::path wallet_dir = gArgs.GetArg("-walletdir", "");
            if (!fs::exists(wallet_dir)) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" does not exist"), wallet_dir.string()));
            } else if (!fs::is_directory(wallet_dir)) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" is not a directory"), wallet_dir.string()));
            } else if (!wallet_dir.is_absolute()) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" is a relative path"), wallet_dir.string()));
            }
        }

-   检查所有的钱包文件。

    首先，确保钱包不存在名称相同的。然后，调用 `CWallet::Verify` 方法检查钱包的路径。

    代码如下：

        for (const auto& wallet_file : wallet_files) {
            fs::path wallet_path = fs::absolute(wallet_file, GetWalletDir());

            if (!wallet_paths.insert(wallet_path).second) {
                return InitError(strprintf(_("Error loading wallet %s. Duplicate -wallet filename specified."), wallet_file));
            }

            std::string error_string;
            std::string warning_string;
            bool verify_success = CWallet::Verify(wallet_file, salvage_wallet, error_string, warning_string);
            if (!error_string.empty()) InitError(error_string);
            if (!warning_string.empty()) InitWarning(warning_string);
            if (!verify_success) return false;
        }
