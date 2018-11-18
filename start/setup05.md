#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第5步，验证钱包数据库完整性（`src/init.cpp::AppInitMain()`）

调用钱包接口的 `Verify` 方法，验证钱包数据库。实现类为 `wallet/init.cpp` 文件中的 `WalletInit` ，方法处理流程如下：

1.  检查启动参数是否禁止钱包 `-disablewallet`。如果是，则直接返回。

        if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
            return true;
        }

2.  如果启动参数指定了钱包路径 `-walletdir`，则检查钱包数据库目录是否存在，是否为目录、且是否为常规的的路径。

        if (gArgs.IsArgSet("-walletdir")) {
            fs::path wallet_dir = gArgs.GetArg("-walletdir", "");
            boost::system::error_code error;
            fs::path canonical_wallet_dir = fs::canonical(wallet_dir, error);
            if (error || !fs::exists(wallet_dir)) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" does not exist"), wallet_dir.string()));
            } else if (!fs::is_directory(wallet_dir)) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" is not a directory"), wallet_dir.string()));
            } else if (!wallet_dir.is_absolute()) {
                return InitError(strprintf(_("Specified -walletdir \"%s\" is a relative path"), wallet_dir.string()));
            }
            gArgs.ForceSetArg("-walletdir", canonical_wallet_dir.string());
        }

3.  从启动参数中取得所有的钱包名称。

        std::vector<std::string> wallet_files = gArgs.GetArgs("-wallet");

4.  根据启动 `-salvagewallet` 和用户指定的钱包数量设置变量 `salvage_wallet`。

        bool salvage_wallet = gArgs.GetBoolArg("-salvagewallet", false) && wallet_files.size() <= 1;

5.  `for` 循环检查用户提供的所有的钱包，此处至少有一个默认钱包，所以肯定会至少循环一次。

    如果用户没有指定启动参数 `-wallet`，则在第三步的第21小步中，调用钱包初始接口对象的 `ParameterInteraction` 方法时，设置启动参数 `-wallet`默认为空字符串，从而在本步时至少创建一个钱包名称为空的默认钱包。
    
    -   根据钱包名称和钱包存放目录，求出钱包的绝对路径。

            fs::path wallet_path = fs::absolute(wallet_file, GetWalletDir());

    -   如果某个钱包的名字有重复，则返回初始化错误。

            if (!wallet_paths.insert(wallet_path).second) {
                return InitError(strprintf(_("Error loading wallet %s. Duplicate -wallet filename specified."), wallet_file));
            }

    -   调用 `CWallet::Verify` 方法，检查钱包。如果出错，则返回错误。

            std::string error_string;
            std::string warning_string;
            bool verify_success = CWallet::Verify(wallet_file, salvage_wallet, error_string, warning_string);
            if (!error_string.empty()) InitError(error_string);
            if (!warning_string.empty()) InitWarning(warning_string);
            if (!verify_success) return false;

        这个方法主要是检查钱包的路径方面的。
