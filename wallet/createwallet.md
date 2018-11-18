#   创建钱包流程

比特币用户最关心除了交易之外就是地址、钱包、私钥了，交易、地址、钱包、私钥这些不同概念之间具有内在的联系，要了解交易必须先要了解地址、钱包、私钥这几个概念，从本章开始，我们开始学习这一部分内容。


前面我们提到 RPC 的概念，RPC 是 remote process call 这个过程的缩写，也就是远程过程调用。比特币核心提供了很多 RPC 来供客户端调用，其中一个就是我们这里要讲的 `createwallet` 创建一个钱包，通过这个 RPC ，我们就可以生成一个新的钱包。

`createwallet` RPC 可以接收两个参数，第一个钱包名称，第二个是否禁止私钥。第一个参数是必填参数，第二个是可选参数，默认为假。

下面我们通过这个 RPC 来看下怎么生成一个钱包。

1.  从参数中取得钱包的名称。

        std::string wallet_name = request.params[0].get_str();
        std::string error;
        std::string warning;

2.  如果提供了第2个参数则取得是否禁止私钥。

        bool disable_privatekeys = false;
        if (!request.params[1].isNull()) {
            disable_privatekeys = request.params[1].get_bool();
        }

3.  通过钱包名称加上其保存的路径来判断钱包名称是否已经存在。

        fs::path wallet_path = fs::absolute(wallet_name, GetWalletDir());
        if (fs::symlink_status(wallet_path).type() != fs::file_not_found) {
          throw JSONRPCError(RPC_WALLET_ERROR, "Wallet " + wallet_name + " already exists.");
        }

4.  检查钱包文件的路径可以创建、不与别的钱包重复、路径是一个目录，同时为了向后兼容，钱包路径要在 `-walletdir` 指定的路径下面。

        if (!CWallet::Verify(wallet_name, false, error, warning)) {
            throw JSONRPCError(RPC_WALLET_ERROR, "Wallet file verification failed: " + error);
        }

5.  创建钱包，如果创建失败，则抛出异常。

        std::shared_ptr<CWallet> const wallet = CWallet::CreateWalletFromFile(wallet_name, fs::absolute(wallet_name, GetWalletDir()), (disable_privatekeys ? (uint64_t)WALLET_FLAG_DISABLE_PRIVATE_KEYS : 0));
        if (!wallet) {
            throw JSONRPCError(RPC_WALLET_ERROR, "Wallet creation failed.");
        }

    `CreateWalletFromFile` 是我们这部分内部的主体，所以放在下面进行详细说明，此处略过不讲。

6.  调用`AddWallet` 方法，把钱包对象加入 `vpwallets` 向量中。

7.  调用钱包对象的 `postInitProcess` 方法，进行一些后置的处理。

8.  返回钱包名字和警告信息组成的对象。


##  1、CreateWalletFromFile 创建钱包入口

这个方法接收 3 个参数，第一个参数是钱包的名称，第二个参数是钱包的绝对路径，第三个参数是钱包的标志。具体逻辑如下：

1.  生成两个变量，一个是钱包文件，一个是钱包的交易元数据，用来在清除交易之后恢复钱包交易。

        const std::string& walletFile = name;
        std::vector<CWalletTx> vWtx;

2.  如果启动参数指定了 `-zapwallettxes`，那么生成一个临时钱包，并调用其 `ZapWalletTx` 方法来清除交易。如果出现错误，则返回空指针。

        if (gArgs.GetBoolArg("-zapwallettxes", false)) {
            std::unique_ptr<CWallet> tempWallet = MakeUnique<CWallet>(name, WalletDatabase::Create(path));
            DBErrors nZapWalletRet = tempWallet->ZapWalletTx(vWtx);
            if (nZapWalletRet != DBErrors::LOAD_OK) {
                InitError(strprintf(_("Error loading %s: Wallet corrupted"), walletFile));
                return nullptr;
            }
        }

    生成钱包时，需要指定钱包文件的路径，从而可以读取钱包中保存的交易信息。

    下面，我们来看下 `ZapWalletTx` 这个方法。方法的主体是生成一个可以访问钱包的数据库的对象，并调用后者的 `ZapWalletTx` 方法来完成清理交易的。在后者的方法定义如下：

        DBErrors WalletBatch::ZapWalletTx(std::vector<CWalletTx>& vWtx)
        {
            std::vector<uint256> vTxHash;
            DBErrors err = FindWalletTx(vTxHash, vWtx);
            if (err != DBErrors::LOAD_OK)
                return err;
            for (const uint256& hash : vTxHash) {
                if (!EraseTx(hash))
                    return DBErrors::CORRUPT;
            }
            return DBErrors::LOAD_OK;
        }

    在上面的方法中，调用 `FindWalletTx` 方法，从钱包数据库中读取所有的交易哈希和交易本身，并生成对应的 `CWalletTx` 对象。然后，遍历所有的交易哈希，调用 `EraseTx` 方法来删除对应的交易。

    为什么要清理钱包数据库中的交易呢？因为有交易费用过低，导致无法打包到区块中，对于这些交易我们需要从钱包文件中清理出去。

3.  生成两个变量，一个是当前时间，一个是表示是否第一次创建钱包。

        int64_t nStart = GetTimeMillis();
        bool fFirstRun = true;

4.  根据钱包名称和绝对路径生成钱包对象。

        std::shared_ptr<CWallet> walletInstance(new CWallet(name, WalletDatabase::Create(path)), ReleaseWallet);

5.  调用钱包对象的 `LoadWallet` 方法，加载钱包。

        DBErrors nLoadWalletRet = walletInstance->LoadWallet(fFirstRun);
 
    如果加载过程出现错误，则进行相应的处理，错误我们这里不讲，具体自己看代码。我们重点看下 `LoadWallet` 方法的逻辑。

    -   重置参数 `fFirstRunRet` 为假。

    -   生成一个可以访问钱包的数据库的对象，从而调用钱包数据库对象的的 `LoadWallet` 方法来加载钱包。从钱包数据库中加载钱包的方法在下面详细说明，此处略过。

            DBErrors nLoadWalletRet = WalletBatch(*database,"cr+").LoadWallet(this);

    -   如果从数据库读取钱包数据结果为 `NEED_REWRITE`，并且数据库重写 `x04pool` 成功，那么设置钱包的内部密钥池、外部密钥池、密钥索引集合等为空。

            if (nLoadWalletRet == DBErrors::NEED_REWRITE)
            {
                if (database->Rewrite("\x04pool"))
                {
                    setInternalKeyPool.clear();
                    setExternalKeyPool.clear();
                    m_pool_key_to_index.clear();
                }
            }

    -   如果 `mapKeys`、`mapCryptedKeys`、`mapWatchKeys`、`setWatchOnly`、`mapScripts` 4个集合为空，且钱包没有设置禁止私钥，那么设置 `fFirstRunRet` 为真。

            fFirstRunRet = mapKeys.empty() && mapCryptedKeys.empty() && mapWatchKeys.empty() && setWatchOnly.empty() && mapScripts.empty() && !IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS);

    -   返回数据库读取的结果。
    
6.  如果启动参数 `-upgradewallet` 为真，或者没有指定启动参数 `-upgradewallet` ，但是第一次运行运行这个钱包，那么进行如下处理：

    -   获取用户指定升级的版本，保存为要升级的版本。

            int nMaxVersion = gArgs.GetArg("-upgradewallet", 0);

    -   如果要升级的版本为0，即用户没有指定具体升级到哪个版本，那么设置要升级的版本等于 `FEATURE_LATEST（目前为 169900）`，即支持 HD 分割，同时调用钱包对象的 `SetMinVersion` 方法，设置钱包最小版本为这个要升级的版本。

            if (nMaxVersion == 0)
            {
                walletInstance->WalletLogPrintf("Performing wallet upgrade to %i\n", FEATURE_LATEST);
                nMaxVersion = FEATURE_LATEST;
                walletInstance->SetMinVersion(FEATURE_LATEST); // permanently upgrade the wallet immediately
            }
            else
                walletInstance->WalletLogPrintf("Allowing wallet upgrade up to %i\n", nMaxVersion);

    -   如果要升级的版本小于钱包当前的版本，则直接返回空指针。

            int nMaxVersion = gArgs.GetArg("-upgradewallet", 0);
            if (nMaxVersion < walletInstance->GetVersion())
            {
                InitError(_("Cannot downgrade wallet"));
                return nullptr;
            }

    -   调用钱包对象的 `SetMinVersion` 方法，设置钱包最大版本为要升级的版本。

            walletInstance->SetMaxVersion(nMaxVersion);

7.  如果指定启动参数 `-upgradewallet` 为真，那么显式升级到 HD 钱包。具体处理如下：

    -   如果钱包不支持 `FEATURE_HD_SPLIT`，并且钱包的版本大于等于 139900 （`FEATURE_HD_SPLIT`）且小于等于 169900 （`FEATURE_PRE_SPLIT_KEYPOOL`），则返回空指针。

            int max_version = walletInstance->nWalletVersion;
            if (!walletInstance->CanSupportFeature(FEATURE_HD_SPLIT) && max_version >=FEATURE_HD_SPLIT && max_version < FEATURE_PRE_SPLIT_KEYPOOL) {
                return nullptr;
            }

    -   如果钱包支持 `FEATURE_HD`，且当前没有启用 HD，那么调用 `SetMinVersion` 方法，设置最小版本为 `FEATURE_HD`，同时调用 `GenerateNewSeed` 方法，生成新的公钥，然后调用 `SetHDSeed` 方法，把新生成的公钥作为 HD 链的随机种子。

            bool hd_upgrade = false;
            bool split_upgrade = false;
            if (walletInstance->CanSupportFeature(FEATURE_HD) && !walletInstance->IsHDEnabled()) {
                walletInstance->SetMinVersion(FEATURE_HD);
                CPubKey masterPubKey = walletInstance->GenerateNewSeed();
                walletInstance->SetHDSeed(masterPubKey);
                hd_upgrade = true;
            }

        `GenerateNewSeed` 方法在下面进行详细说明。

    -   如果钱包支持 `FEATURE_HD_SPLIT`，那么调用 `SetMinVersion` 方法，设置最小版本为 `FEATURE_HD_SPLIT`。如果 `FEATURE_HD_SPLIT` 大于先前的版本，则设置变量 `split_upgrade` 为真。

            if (walletInstance->CanSupportFeature(FEATURE_HD_SPLIT)) {
                walletInstance->WalletLogPrintf("Upgrading wallet to use HD chain split\n");
                walletInstance->SetMinVersion(FEATURE_PRE_SPLIT_KEYPOOL);
                split_upgrade = FEATURE_HD_SPLIT > prev_version;
            }

    -   如果变量 `split_upgrade` 为真，则调用 `MarkPreSplitKeys` 方法，将当前位于密钥池中的所有密钥标记为预拆分。

            if (split_upgrade) {
                walletInstance->MarkPreSplitKeys();
            }

    -   如果是 HD 升级，那么重新生成密钥池。

            if (hd_upgrade) {
                if (!walletInstance->TopUpKeyPool()) {
                    InitError(_("Unable to generate keys"));
                    return nullptr;
                }
            }

        `TopUpKeyPool` 方法在下面进行详细说明。

8.  如果是第一次创建钱包，那么进行下面的处理。

    `fFirstRun` 变量表示是否是第一次创建钱包。系统在运行时，每一次通过 `createwallet` RPC 来创建钱包时，这个变量都是真，即第一次运行，那什么时候这个变量会是假呢？答案是，在钱包创建之后，比特币客户端被关掉并再次启动之后，这个变量就在钱包对象的 `LoadWallet` 方法中被设置为假，即不是第一次运行。

    所以以下逻辑只有在创建默认钱包时才会执行。

    -   设置钱包最小版本为 `FEATURE_LATEST`，当前为 `FEATURE_PRE_SPLIT_KEYPOOL`。

            walletInstance->SetMinVersion(FEATURE_LATEST);

    -   如果创建参数指定不能包含私钥，那么设置钱包这个标记；否则，调用 `GenerateNewSeed` 方法，生成新的随机种子，然后调用 `SetHDSeed` 方法，保存随机种子。

            if ((wallet_creation_flags & WALLET_FLAG_DISABLE_PRIVATE_KEYS)) {
                walletInstance->SetWalletFlag(WALLET_FLAG_DISABLE_PRIVATE_KEYS);
            } else {
                CPubKey seed = walletInstance->GenerateNewSeed();
                walletInstance->SetHDSeed(seed);
            }

        `GenerateNewSeed` 方法，下面进行讲解。

    -   如果创建参数没有指定不能包含私钥，那么填充密钥池。如果失败，即不能生成初始密钥，则返回空指针。

            if (!walletInstance->IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS) && !walletInstance->TopUpKeyPool()) {
                return nullptr;
            }

        `TopUpKeyPool` 方法，下面进行讲解。

    -   刷新到数据库。

            walletInstance->ChainStateFlushed(chainActive.GetLocator());

9.  否则，如果创建参数指定不能包含私钥，那么返回 NULL。

        else if (wallet_creation_flags & WALLET_FLAG_DISABLE_PRIVATE_KEYS) {
            InitError(strprintf(_("Error loading %s: Private keys can only be disabled during creation"), walletFile));
            return NULL;
        }

10.  如果指定了启动参数 `-addresstype`，但是解析失败，则返回空指针。

            if (!gArgs.GetArg("-addresstype", "").empty() && !ParseOutputType(gArgs.GetArg("-addresstype", ""), walletInstance->m_default_address_type)) {
                InitError(strprintf("Unknown address type '%s'", gArgs.GetArg("-addresstype", "")));
                return nullptr;
            }

11. 如果指定了启动参数 `-changetype`，但是解析失败，则返回空指针。

        if (!gArgs.GetArg("-changetype", "").empty() && !ParseOutputType(gArgs.GetArg("-changetype", ""), walletInstance->m_default_change_type)) {
            InitError(strprintf("Unknown change type '%s'", gArgs.GetArg("-changetype", "")));
            return nullptr;
        }

12. 如果设置了最小交易费用，则解析并设置钱包的最小交易费用。

        if (gArgs.IsArgSet("-mintxfee")) {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-mintxfee", ""), n) || 0 == n) {
                return nullptr;
            }
            walletInstance->m_min_fee = CFeeRate(n);
        }

13. 根据不同网络取得是否启用回退费用。如果启动参数设置了 `-fallbackfee` 用以在估算费用不足时将使用的费率，那么就解析设置的费率，如果解析错误，则打印错误日志，并返回空指针，如果解析OK，但是大于规定的最大值，则打印警告日志。如果两者都没有问题，则设置钱包的回退费率。

        walletInstance->m_allow_fallback_fee = Params().IsFallbackFeeEnabled();
        if (gArgs.IsArgSet("-fallbackfee")) {
            CAmount nFeePerK = 0;
            if (!ParseMoney(gArgs.GetArg("-fallbackfee", ""), nFeePerK)) {
                InitError(strprintf(_("Invalid amount for -fallbackfee=<amount>: '%s'"), gArgs.GetArg("-fallbackfee", "")));
                return nullptr;
            }
            if (nFeePerK > HIGH_TX_FEE_PER_KB) {
                InitWarning(AmountHighWarn("-fallbackfee") + " " +
                            _("This is the transaction fee you may pay when fee estimates are not available."));
            }
            walletInstance->m_fallback_fee = CFeeRate(nFeePerK);
            walletInstance->m_allow_fallback_fee = nFeePerK != 0; //disable fallback fee in case value was set to 0, enable if non-null value
        }

14. 如果启动参数设置了 `-discardfee` 用以规定在费用小于多少时舍弃，那么就解析设置的费率，如果解析错误，则打印错误日志，并返回空指针，如果解析OK，但是大于规定的最大值，则打印警告日志。如果两者都没有问题，则设置钱包的丢弃费率。

        if (gArgs.IsArgSet("-discardfee")) {
            CAmount nFeePerK = 0;
            if (!ParseMoney(gArgs.GetArg("-discardfee", ""), nFeePerK)) {
                InitError(strprintf(_("Invalid amount for -discardfee=<amount>: '%s'"), gArgs.GetArg("-discardfee", "")));
                return nullptr;
            }
            if (nFeePerK > HIGH_TX_FEE_PER_KB) {
                InitWarning(AmountHighWarn("-discardfee") + " " +
                            _("This is the transaction fee you may discard if change is smaller than dust at this level"));
            }
            walletInstance->m_discard_rate = CFeeRate(nFeePerK);
        }

15. 如果启用参数设置了 `-paytxfee` 指定交易费用，那么就解析设置的费率，如果解析错误，则打印错误日志，并返回空指针，如果解析OK，但是大于规定的最大值，则打印警告日志。如果两者都没有问题，则设置钱包的交易费用。如果交易费用小于规定的最小值，则认为交易费用为为，则打印警告日志，并返回空指针。

        if (gArgs.IsArgSet("-paytxfee")) {
          CAmount nFeePerK = 0;
          if (!ParseMoney(gArgs.GetArg("-paytxfee", ""), nFeePerK)) {
              InitError(AmountErrMsg("paytxfee", gArgs.GetArg("-paytxfee", "")));
              return nullptr;
          }
          if (nFeePerK > HIGH_TX_FEE_PER_KB) {
              InitWarning(AmountHighWarn("-paytxfee") + " " +
                          _("This is the transaction fee you will pay if you send a transaction."));
          }
          walletInstance->m_pay_tx_fee = CFeeRate(nFeePerK, 1000);
          if (walletInstance->m_pay_tx_fee < ::minRelayTxFee) {
              InitError(strprintf(_("Invalid amount for -paytxfee=<amount>: '%s' (must be at least %s)"),
                  gArgs.GetArg("-paytxfee", ""), ::minRelayTxFee.ToString()));
              return nullptr;
          }
        }

16. 设置交易确认的平均区块数（默认为6）、是否发送未确认的变更、是否启用 full-RBF opt-in。

        walletInstance->m_confirm_target = gArgs.GetArg("-txconfirmtarget", DEFAULT_TX_CONFIRM_TARGET);
        walletInstance->m_spend_zero_conf_change = gArgs.GetBoolArg("-spendzeroconfchange", DEFAULT_SPEND_ZEROCONF_CHANGE);
        walletInstance->m_signal_rbf = gArgs.GetBoolArg("-walletrbf", DEFAULT_WALLET_RBF);

17. 接下来填充密钥池，如果钱包被锁定，则不进行任何操作。

        walletInstance->TopUpKeyPool();

    `TopUpKeyPool` 下面进行详述讲解，此处不提。

18. 获取区块链的创世区块索引。

        CBlockIndex *pindexRescan = chainActive.Genesis();

    `pindexRescan` 表示需要重新扫描区块链的区块，默认从创世区块开始，下面会进行重新设置。

19. 如果没有指定重新扫描区块链，或指定不扫描，那么进行下面的操作。

    -   生成一个可以访问钱包数据库的 `batch` 对象。

            WalletBatch batch(*walletInstance->database);

    -   调用 `batch` 对象的 `ReadBestBlock` 方法，从数据库读取最佳区块，即关键字为 `bestblock` 的区块。如果可以找到这个区块，那么调用 `FindForkInGlobalIndex` 方法，返回分叉的区块。

            CBlockLocator locator;
            if (batch.ReadBestBlock(locator))
                pindexRescan = FindForkInGlobalIndex(chainActive, locator);

        其实最佳区块不是一个区块，而是一个区块定位器，它描述了块链中到另一个节点的位置，这样如果另一个节点没有相同的分支，它就可以找到最近的公共中继。

    `FindForkInGlobalIndex` 方法根据区块定位器中包含的区块哈希在区块索引集合 `mapBlockIndex` 中查找对应的区块。如果这个区块索引存在，进一步如果当前区块链中包含这个区块索引，则返回这个区块索引，否则如果这个区块索引的祖先是当前区块链的顶部区块，则返回当前区块链的顶部区块。最后，如果找不到这样的区块，则返回当前区块链的创世区块。

        CBlockIndex* FindForkInGlobalIndex(const CChain& chain, const CBlockLocator& locator)
        {
            AssertLockHeld(cs_main);
            for (const uint256& hash : locator.vHave) {
                CBlockIndex* pindex = LookupBlockIndex(hash);
                if (pindex) {
                    if (chain.Contains(pindex))
                        return pindex;
                    if (pindex->GetAncestor(chain.Height()) == chain.Tip()) {
                        return chain.Tip();
                    }
                }
            }
            return chain.Genesis();
        }

20. 设置钱包最后一个处理的区块为当前区块链顶部的区块。

        walletInstance->m_last_block_processed = chainActive.Tip();

21. 如果当前区块链顶部的区块存在，且不等于前一步中我们找到的 `pindexRescan` 对应的区块，那么进行下面的处理：

    -   如果当前是修剪模式，从区块链顶部的区块开始向下遍历，一直找到某个区块的前一区块的数据不在区块数据库文件或，或者前一个区块的交易数为0，或者这个区块是 `pindexRescan`。如果最终找到的区块不是 `pindexRescan`，那么打印错误消息，并返回空指针。

            if (fPruneMode)
            {
                CBlockIndex *block = chainActive.Tip();
                while (block && block->pprev && (block->pprev->nStatus & BLOCK_HAVE_DATA) && block->pprev->nTx > 0 && pindexRescan != block)
                    block = block->pprev;

                if (pindexRescan != block) {
                    InitError(_("Prune: last wallet synchronisation goes beyond pruned data. You need to -reindex (download the whole blockchain again in case of pruned node)"));
                    return nullptr;
                }
            }

    -   从 `pindexRescan` 区块开始向区块链顶部遍历，找到第一个区块创建时间小于钱包创建时间与 `TIMESTAMP_WINDOW` 之差的区块。`TIMESTAMP_WINDOW` 当前规定为 2个小时。

            while (pindexRescan && walletInstance->nTimeFirstKey && (pindexRescan->GetBlockTime() < (walletInstance->nTimeFirstKey - TIMESTAMP_WINDOW))) {
                pindexRescan = chainActive.Next(pindexRescan);
            }

    -   生成一个 `WalletRescanReserver` 对象，并调用其 `reserve` 方法。如果该方法返回假，则打印错误日志，并返回空指针。否则，调用钱包对象的 `ScanForWalletTransactions` 方法，扫描钱包的所有交易。

        `reserve` 方法内部检查钱包的 `fScanningWallet` 属性，如果这个属性已经为真，那么直接返回假；否则，设置其为真，并设置变量 `m_could_reserve` 也为真，然后返回真。具体代码如下：

            bool reserve()
            {
                assert(!m_could_reserve);
                std::lock_guard<std::mutex> lock(m_wallet->mutexScanning);
                if (m_wallet->fScanningWallet) {
                    return false;
                }
                m_wallet->fScanningWallet = true;
                m_could_reserve = true;
                return true;
            }

        `ScanForWalletTransactions` 方法，具体处理逻辑如下：

        -   首先，进行些变量初始化及校验，不详述。

                int64_t nNow = GetTime();
                const CChainParams& chainParams = Params();
                assert(reserver.isReserved());
                if (pindexStop) {
                    assert(pindexStop->nHeight >= pindexStart->nHeight);
                }
                CBlockIndex* pindex = pindexStart;
                CBlockIndex* ret = nullptr;

        -   调用 `GuessVerificationProgress` 方法，预估验证的进度。

                fAbortRescan = false;
                CBlockIndex* tip = nullptr;
                double progress_begin;
                double progress_end;
                {
                    LOCK(cs_main);
                    progress_begin = GuessVerificationProgress(chainParams.TxData(), pindex);
                    if (pindexStop == nullptr) {
                        tip = chainActive.Tip();
                        progress_end = GuessVerificationProgress(chainParams.TxData(), tip);
                    } else {
                        progress_end = GuessVerificationProgress(chainParams.TxData(), pindexStop);
                    }
                }

        -   只要 `pindex` 不为空，且没有终止当前的扫描，且没有收到关闭的请求，就沿着区块链向栈顶遍历，从硬盘上读取区块，如果可以读取到区块，并且当前活跃区块链包含当前的区块，那么从这个区块中同步所有的交易，如果当前活跃区块链不包含当前的区块，那么设置当前区块索引为退出索引，并且退出循环；如果不能从硬盘中读取，同样设置当前区块索引为退出索引。如果当前区块等于退出区块的索引，那么退出循环。

                double progress_current = progress_begin;
                while (pindex && !fAbortRescan && !ShutdownRequested())
                {
                    if (GetTime() >= nNow + 60) {
                        nNow = GetTime();
                    }
                    CBlock block;
                    if (ReadBlockFromDisk(block, pindex, Params().GetConsensus())) {
                        LOCK2(cs_main, cs_wallet);
                        if (pindex && !chainActive.Contains(pindex)) {
                            ret = pindex;
                            break;
                        }
                        for (size_t posInBlock = 0; posInBlock < block.vtx.size(); ++posInBlock) {
                            SyncTransaction(block.vtx[posInBlock], pindex, posInBlock, fUpdate);
                        }
                    } else {
                        ret = pindex;
                    }
                    if (pindex == pindexStop) {
                        break;
                    }
                    {
                        LOCK(cs_main);
                        pindex = chainActive.Next(pindex);
                        progress_current = GuessVerificationProgress(chainParams.TxData(), pindex);
                        if (pindexStop == nullptr && tip != chainActive.Tip()) {
                            tip = chainActive.Tip();
                            progress_end = GuessVerificationProgress(chainParams.TxData(), tip);
                        }
                    }
                }

        -   返回退出区块索引。

    -   调用钱包对象的 `ChainStateFlushed` 方法，把区块定位器作为最佳区块保存到钱包数据库中。

            walletInstance->ChainStateFlushed(chainActive.GetLocator());

    -   调用钱包数据库对象的 `IncrementUpdateCounter` 方法，增加数据库对象的更新次数。

    -   如果启动参数指定了 `-zapwallettxes`，且等于1，那么需要重新把钱包的元数据保存到钱包数据库中。钱包交易前面取出来保存在 `vWtx` 变量中。

            if (gArgs.GetBoolArg("-zapwallettxes", false) && gArgs.GetArg("-zapwallettxes", "1") != "2")
            {
                WalletBatch batch(*walletInstance->database);

                for (const CWalletTx& wtxOld : vWtx)
                {
                    uint256 hash = wtxOld.GetHash();
                    std::map<uint256, CWalletTx>::iterator mi = walletInstance->mapWallet.find(hash);
                    if (mi != walletInstance->mapWallet.end())
                    {
                        const CWalletTx* copyFrom = &wtxOld;
                        CWalletTx* copyTo = &mi->second;
                        copyTo->mapValue = copyFrom->mapValue;
                        copyTo->vOrderForm = copyFrom->vOrderForm;
                        copyTo->nTimeReceived = copyFrom->nTimeReceived;
                        copyTo->nTimeSmart = copyFrom->nTimeSmart;
                        copyTo->fFromMe = copyFrom->fFromMe;
                        copyTo->nOrderPos = copyFrom->nOrderPos;
                        batch.WriteTx(*copyTo);
                    }
                }
            }

22. 调用全局方法 `RegisterValidationInterface` 方法，注册钱包通过 `boost::bind` 方法返回的绑定方法作为信号处理器。

    钱包对象继承了 `CValidationInterface` 接口，实现了以下几个方法：

    -   TransactionAddedToMempool

    -   BlockConnected

    -   BlockDisconnected

    -   TransactionRemovedFromMempool

    -   ResendWalletTransactions

23.  调用钱包对象的 `SetBroadcastTransactions` 方法，根据启动参数设置是否广播交易。

            walletInstance->SetBroadcastTransactions(gArgs.GetBoolArg("-walletbroadcast", DEFAULT_WALLETBROADCAST));

    方法内部设置钱包对象的 `fBroadcastTransactions` 属性值。

24. 返回钱包对象。


##  重要方法

### 2.1、GenerateNewSeed

这个方法主要用来生成私钥/公钥，在生成公钥后，调用钱包对象的 `SetHDSeed` 方法，根据公钥生成一个 `CHDChain` 对象，把公钥经过 SHA256、RIPEMD160 两次哈希返回后的 20个字节 180 位字符串作为它的 `seed_id` 属性，从而以后可以衍生更的扩展公钥。

    void CWallet::SetHDSeed(const CPubKey& seed)
    {
        LOCK(cs_wallet);
        CHDChain newHdChain;
        newHdChain.nVersion = CanSupportFeature(FEATURE_HD_SPLIT) ? CHDChain::VERSION_HD_CHAIN_SPLIT : CHDChain::VERSION_HD_BASE;
        newHdChain.seed_id = seed.GetID();
        SetHDChain(newHdChain, false);
    }


在创建钱包的过程中，本方法及下面的 `TopUpKeyPool` 方法，可能会被调用两次，一次在第一次创建钱包时，另一次在用户明确升级，且钱包支持 HD，但 HD 没启用的情况下。

下面，我们开始正式讲解这个方法。

1.  首先，生成并调用私钥的 `MakeNewKey` 来初始化私钥。

        CKey key;
        key.MakeNewKey(true);

2.  然后，调用 `DeriveNewSeed` 方法，生成并返回私钥对应的公钥。

        return DeriveNewSeed(key);

我们先来看 `MakeNewKey` 这个方法。这个方法比较简单，但是非常重要，因为正是这个方法，生成了真正意义上的私钥。

    void CKey::MakeNewKey(bool fCompressedIn) {
        do {
            GetStrongRandBytes(keydata.data(), keydata.size());
        } while (!Check(keydata.data()));
        fValid = true;
        fCompressed = fCompressedIn;
    }

`GetStrongRandBytes` 方法，正是生成私钥的过程。这个方法的执行逻辑如下：

1.  生成一个 SHA512对象和一个无符号字符数组。

        CSHA512 hasher;
        unsigned char buf[64];

2.  首先，调用 `RandAddSeedPerfmon` 方法，通过 OpenSSL's RNG 生成随机数，并进行 SHA512 哈希。

        RandAddSeedPerfmon();
        GetRandBytes(buf, 32);
        hasher.Write(buf, 32);

3.  然后，调用 `GetOSRand` 方法，通过 OS RNG 生成随机数，并进行 SHA512 哈希。

        GetOSRand(buf);
        hasher.Write(buf, 32);

4.  最后，如果 HW 可用，通过 HW 生成随机数，并进行 SHA512 哈希。

        if (GetHWRand(buf)) {
            hasher.Write(buf, 32);
        }

5.  接下来，合并并更新状态。

        {
            WAIT_LOCK(cs_rng_state, lock);
            hasher.Write(rng_state, sizeof(rng_state));
            hasher.Write((const unsigned char*)&rng_counter, sizeof(rng_counter));
            ++rng_counter;
            hasher.Finalize(buf);
            memcpy(rng_state, buf + 32, 32);
        }

6.  然后，把 `buf` 数组中的内容拷贝到输出参数中，并清空数组中的内容。

        memcpy(out, buf, num);
        memory_cleanse(buf, 64);

`Check` 方法内部通过调用 `secp256k1_ec_seckey_verify` 方法，检查私钥的数据是否满足要求，后者正是椭圆曲线算法相关的。

接下来，我们来看 `DeriveNewSeed` 方法是如何生成公钥的。方法执行逻辑如下：

1.  生成当前时间，并用当前时间初始化蜜钥元数据。

        int64_t nCreationTime = GetTime();
        CKeyMetadata metadata(nCreationTime);

2.  调用私钥的 `GetPubKey` 方法，生成公钥。

    方法内部正是通过椭圆曲线算法来生成私钥对应的公钥。代码如下，有兴趣的读者可以自行研究。

        assert(fValid);
        secp256k1_pubkey pubkey;
        size_t clen = CPubKey::PUBLIC_KEY_SIZE;
        CPubKey result;
        int ret = secp256k1_ec_pubkey_create(secp256k1_context_sign, &pubkey, begin());
        assert(ret);
        secp256k1_ec_pubkey_serialize(secp256k1_context_sign, (unsigned char*)result.begin(), &clen, &pubkey, fCompressed ? SECP256K1_EC_COMPRESSED : SECP256K1_EC_UNCOMPRESSED);
        assert(result.size() == clen);
        assert(result.IsValid());
        return result;

3.  执行 `assert(key.VerifyPubKey(seed));` 方法，验证公钥确实有私钥相匹配。

4.  设置蜜钥元数据的两个属性。

        metadata.hdKeypath     = "s";
        metadata.hd_seed_id = seed.GetID();

    `GetID` 方法，用公钥的数据经过 SHA256、RIPEMD160 两次哈希后生成的字符串来生成一个 `CKeyID` 对象。

5.  把密钥元数据保存在 `mapKeyMetadata` 集合中。

        mapKeyMetadata[seed.GetID()] = metadata;

6.  调用 `AddKeyPubKey` 方法，把私钥/公钥及元数据保存到数据库。如果出错，抛出一个异常。

        if (!AddKeyPubKey(key, seed))
            throw std::runtime_error(std::string(__func__) + ": AddKeyPubKey failed");

    `AddKeyPubKey` 方法生成一个访问数据库的对象，然后调用钱包对象的 `AddKeyPubKeyWithDB` 方法来保存密钥数据。后者的执行流程如下：

    -   如果变量 `encrypted_batch` 为空，那么设置变量 `needsDB` 为真。如果后者为真，那么设置 `encrypted_batch` 属性为参数指定的对象。

            bool needsDB = !encrypted_batch;
            if (needsDB) {
                encrypted_batch = &batch;
            }

    -   调用 `CCryptoKeyStore::AddKeyPubKey` 方法，保存私钥和公钥。如果不成功，进一步，如果变量 `needsDB` 为真，则设置变量 `encrypted_batch` 为假。

            if (!CCryptoKeyStore::AddKeyPubKey(secret, pubkey)) {
                if (needsDB) encrypted_batch = nullptr;
                return false;
            }

        `CCryptoKeyStore::AddKeyPubKey` 方法内部执行逻辑如下：

        -   如果没有加密，那么调用 `CBasicKeyStore::AddKeyPubKey` 来保存私钥和公钥，然后返回。

                if (!IsCrypted()) {
                    return CBasicKeyStore::AddKeyPubKey(key, pubkey);
                }

            `IsCrypted` 方法，初始化时是没有加密的，只有在执行加锁、解锁、添加加密密钥的情况下才会是加密的。当前情景是没有加密的，所以执行基类的方法来保存私钥和公钥。

            `CBasicKeyStore::AddKeyPubKey` 方法，首先把私钥保存在 `mapKeys` 集合中，然后调用 `ImplicitlyLearnRelatedKeyScripts` 方法来处理脚本，因为公钥默认是压缩的，所以在方法内会生成脚本，并把脚本保存在 `mapScripts` 集合中。

                bool CBasicKeyStore::AddKeyPubKey(const CKey& key, const CPubKey &pubkey)
                {
                    LOCK(cs_KeyStore);
                    mapKeys[pubkey.GetID()] = key;
                    ImplicitlyLearnRelatedKeyScripts(pubkey);
                    return true;
                }

        -   如果是锁定的，那么返回假。

                if (IsLocked()) {
                    return false;
                }

            `IsLocked` 方法，首先调用 `IsCrypted` 方法，确定是否是加密的，如果不是加密的，则直接返回假；否则，把 `vMasterKey` 集合清空。

                bool CCryptoKeyStore::IsLocked() const
                {
                    if (!IsCrypted()) {
                        return false;
                    }
                    LOCK(cs_KeyStore);
                    return vMasterKey.empty();
                }

        -   接下来，生成加密私钥。

                std::vector<unsigned char> vchCryptedSecret;
                CKeyingMaterial vchSecret(key.begin(), key.end());
                if (!EncryptSecret(vMasterKey, vchSecret, pubkey.GetHash(), vchCryptedSecret)) {
                    return false;
                }

        -   最后，调用 `AddCryptedKey` 方法，保存加密私钥，并返回

                if (!AddCryptedKey(pubkey, vchCryptedSecret)) {
                    return false;
                }
                return true;

            `AddCryptedKey` 方法，首先确定是否是加密的，如果没有加密则返回假。接下来，把私钥保存在 `mapCryptedKeys` 集合中，然后调用 `ImplicitlyLearnRelatedKeyScripts` 方法来处理脚本，因为公钥默认是压缩的，所以在方法内会生成脚本，并把脚本保存在 `mapScripts` 集合中。

                bool CCryptoKeyStore::AddCryptedKey(const CPubKey &vchPubKey, const std::vector<unsigned char> &vchCryptedSecret)
                {
                    LOCK(cs_KeyStore);
                    if (!SetCrypted()) {
                        return false;
                    }
                    mapCryptedKeys[vchPubKey.GetID()] = make_pair(vchPubKey, vchCryptedSecret);
                    ImplicitlyLearnRelatedKeyScripts(vchPubKey);
                    return true;
                }

    -   如果变量 `needsDB` 为真，则设置变量 `encrypted_batch` 为空指针。

    -   调用 `GetScriptForDestination` 方法，获得公钥对应的脚本。

            CScript script;
            script = GetScriptForDestination(pubkey.GetID());

    -   调用 `HaveWatchOnly` 方法，检查 `setWatchOnly` 集合中是否有这个脚本。如果有，则调用 `RemoveWatchOnly` 方法，清除公钥对应的脚本。

            if (HaveWatchOnly(script)) {
                RemoveWatchOnly(script);
            }

        `RemoveWatchOnly` 方法执行逻辑如下：

        -   调用 `CCryptoKeyStore::RemoveWatchOnly` 方法来移除脚本。方法内部执行逻辑如下：从 `setWatchOnly` 集合中移除对应的脚本；然后，调用 `ExtractPubKey` 方法，通过脚本来解析出公钥，如果可以得到公钥，则从 `mapWatchKeys` 集合中移除对应的公钥；最后，返回真。

        -   调用数据库访问对象的 `EraseWatchOnly` 方法，从数据库中移除 `watchmeta` 和 `watchs` 对应的数据。

        -   返回真。

    -   调用 `GetScriptForRawPubKey` 方法，从原始公钥获得脚本。

            script = GetScriptForRawPubKey(pubkey);

    -   调用 `HaveWatchOnly` 方法，检查 `setWatchOnly` 集合中是否有这个脚本。如果有，则调用 `RemoveWatchOnly` 方法，清除公钥对应的脚本。

        方法执行逻辑如上所述，这里不讲。

    -   如果不是加密的，则调用数据库的访问对象的 `WriteKey` 方法，把私钥和公钥及密钥元数据写入数据库。

        在这个方法中，首先，以 `keymeta` 为键，保存对应的元数据；然后，把公钥/私钥保存到向量中，以 `key` 为键，保存对应的公钥/私钥。

    -   最后，返回真。


### 2.2、TopUpKeyPool

本方法在第一次创建时会执行，在升级钱包到 HD 时，也会执行。它被用来填充密钥池 keypool。下面我们来看下方法的执行。

1.  如果标志禁止私钥，则返回假。

        if (IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS)) {
            return false;
        }

2.  如果钱包被锁定，则返回假。

        if (IsLocked())
            return false;

3.  如果参数 `kpSize` 大于0，则设置变量 `nTargetSize` 为 `kpSize`；否则，设置为用户指定的值，或默认的 1000。

        unsigned int nTargetSize;
        if (kpSize > 0)
            nTargetSize = kpSize;
        else
            nTargetSize = std::max(gArgs.GetArg("-keypool", DEFAULT_KEYPOOL_SIZE), (int64_t) 0);

4.  计算内部、外部可用的密钥数量。

        int64_t missingExternal = std::max(std::max((int64_t) nTargetSize, (int64_t) 1) - (int64_t)setExternalKeyPool.size(), (int64_t) 0);
        int64_t missingInternal = std::max(std::max((int64_t) nTargetSize, (int64_t) 1) - (int64_t)setInternalKeyPool.size(), (int64_t) 0);

    在用户不指定的情况下，并且是第一次，因为 `setExternalKeyPool`、`setInternalKeyPool` 这两个集合都为空，所以 `missingExternal`、`missingInternal` 两个变量的值都为 1000。

5.  如果不支持 HD 钱包，或者不支持 HD 分割，那么设置变量 `missingInternal` 为0。

        if (!IsHDEnabled() || !CanSupportFeature(FEATURE_HD_SPLIT))
        {
            missingInternal = 0;
        }

6.  生成访问数据库的对象。

        bool internal = false;
        WalletBatch batch(*database);

7.  进行 `for (int64_t i = missingInternal + missingExternal; i--;)` 循环。

    -   如果 `i` 小于 `missingInternal` ，则设置变量 `internal` 为真。

            if (i < missingInternal) {
                internal = true;
            }

    -   设置当前索引。

            int64_t index = ++m_max_keypool_index;

    -   生成公钥对象。

            CPubKey pubkey(GenerateNewKey(batch, internal));

        `GenerateNewKey` 方法用来生成公钥。我们来看下这个方法的执行流程。

        -   生成表示创建的公钥是否为压缩的变量 `fCompressed`。在 0.6 版本之后的公钥默认都是压缩的。

                bool fCompressed = CanSupportFeature(FEATURE_COMPRPUBKEY);

        -   创建私钥对象和密钥元数据对象。这时候私钥和密钥元数据对象都没有经过设置，还不能成为真正的私钥和密钥元数据，只有经过下面两个方法中的某一个处理之后，才变成真正可用的对象。

                CKey secret;
                int64_t nCreationTime = GetTime();
                CKeyMetadata metadata(nCreationTime);

        -   如果钱包支持 HD，那么调用 `DeriveNewChildKey` 方法来衍生子私钥，否则，调用 `MakeNewKey` 方法来生成私钥。

                if (IsHDEnabled()) {
                    DeriveNewChildKey(batch, metadata, secret, (CanSupportFeature(FEATURE_HD_SPLIT) ? internal : false));
                } else {
                    secret.MakeNewKey(fCompressed);
                }

            `IsHDEnabled` 是通过私钥对象的 `hdChain.seed_id` 对象是否为空来判断的，因为在前面生成钱包私钥之后，就设置了这个方法，所以这个方法一定返回真。

            `MakeNewKey` 这个方法，我们在前面创建钱包的私钥时已经看到过，`DeriveNewChildKey` 这个方法我们在下面进行重点讲解，这里暂且略过。

        -   如果变量 `fCompressed` 为真，则调用 `SetMinVersion` 方法，设置最小版本为 `FEATURE_COMPRPUBKEY`，支持压缩公钥的版本。

                if (fCompressed) {
                    SetMinVersion(FEATURE_COMPRPUBKEY);
                }

        -   调用私钥的 `GetPubKey` 方法，返回对应的公钥。

            方法内部使用椭圆曲线算法求得公钥。具体不细讲，读者自行看代码。

                CPubKey CKey::GetPubKey() const {
                    assert(fValid);
                    secp256k1_pubkey pubkey;
                    size_t clen = CPubKey::PUBLIC_KEY_SIZE;
                    CPubKey result;
                    int ret = secp256k1_ec_pubkey_create(secp256k1_context_sign, &pubkey, begin());
                    assert(ret);
                    secp256k1_ec_pubkey_serialize(secp256k1_context_sign, (unsigned char*)result.begin(), &clen, &pubkey, fCompressed ? SECP256K1_EC_COMPRESSED : SECP256K1_EC_UNCOMPRESSED);
                    assert(result.size() == clen);
                    assert(result.IsValid());
                    return result;
                }

        -   把密钥元数据加入 `mapKeyMetadata` 集合中。

                mapKeyMetadata[pubkey.GetID()] = metadata;

        -   调用 `UpdateTimeFirstKey` 方法，更新 `nTimeFirstKey` 属性。

                UpdateTimeFirstKey(nCreationTime);
                
        -   调用 `AddKeyPubKeyWithDB` 方法，把私钥和公钥保存到数据库中。这个方法在前面已经讲过，这里再讲了。

                if (!AddKeyPubKeyWithDB(batch, secret, pubkey)) {
                    throw std::runtime_error(std::string(__func__) + ": AddKey failed");
                }

        -   返回公钥。

    -   用公钥生成一个密钥池实体 `CKeyPool` 对象，并调用访问钱包数据库对象的 `WritePool` 方法，以 `pool` 为键把密钥池实体对象写入数据库。

            if (!batch.WritePool(index, CKeyPool(pubkey, internal))) {
                throw std::runtime_error(std::string(__func__) + ": writing generated key failed");
            }

    -   如果变量 `internal` 为真，则把索引保存到 `setInternalKeyPool` 集合中，否则，保存到 `setExternalKeyPool` 集合中。

            if (internal) {
                setInternalKeyPool.insert(index);
            } else {
                setExternalKeyPool.insert(index);
            }

    -   把索引保存到 `m_pool_key_to_index` 集合中。

            m_pool_key_to_index[pubkey.GetID()] = index;

8.  返回真。


####    2.2.1、DeriveNewChildKey

这个方法用来衍生子密钥。方法的逻辑如下：

1.  生成相关的变量。

        CKey seed;                     //seed (256bit)
        CExtKey masterKey;             //hd master key
        CExtKey accountKey;            //key at m/0'
        CExtKey chainChildKey;         //key at m/0'/0' (external) or m/0'/1' (internal)
        CExtKey childKey;              //key at m/0'/0'/<n>'

2.  调用 `GetKey` 方法，根据HD 链对象中保存的公钥对象来得到对应的私钥对象。这个私钥是根私钥，也被称为种子私钥。如果获得不到，则抛出异常。

        if (!GetKey(hdChain.seed_id, seed))
            throw std::runtime_error(std::string(__func__) + ": seed not found");

    `GetKey` 方法执行流程如下：

    -   如果钱包没有加密，则调用 `CBasicKeyStore::GetKey` 方法，返回公钥对象的私钥。

            if (!IsCrypted()) {
                return CBasicKeyStore::GetKey(address, keyOut);
            }

        `CBasicKeyStore::GetKey` 方法直接从 `mapKeys` 集合中取出对应的私钥。

        `mapKeys` 集合是我们前面分析 `DeriveNewSeed` 这个方法的第 6 步 `AddKeyPubKey` 这个方法中把私钥/公钥及元数据保存到数据库过程中设置的。

    -   否则，直接从加密集合 `mapCryptedKeys` 中取得对应的私钥。

            CryptedKeyMap::const_iterator mi = mapCryptedKeys.find(address);
            if (mi != mapCryptedKeys.end())
            {
                const CPubKey &vchPubKey = (*mi).second.first;
                const std::vector<unsigned char> &vchCryptedSecret = (*mi).second.second;
                return DecryptKey(vMasterKey, vchCryptedSecret, vchPubKey, keyOut);
            }

    -   如果以上两种情况都不能返回，那么只能返回假了。

3.  调用主私钥（扩展私钥）的 `SetSeed` 方法，设置种子。此时的主私钥只是一个对象，而不能当作真正的私钥来使用，因为内部数据不存在。只有在本方法调用之后，主私钥才能真正用来衍生密钥。

    我们现在来看下 `SetSeed` 方法的逻辑：

    -   首先，生成需要的变量。

            static const unsigned char hashkey[] = {'B','i','t','c','o','i','n',' ','s','e','e','d'};
            std::vector<unsigned char, secure_allocator<unsigned char>> vout(64);

    -   其次，调用 `CHMAC_SHA512` 方法，使用 `HMAC_SHA512` 算法，根据种子私钥生成长度为 512 位的字符串。

            CHMAC_SHA512(hashkey, sizeof(hashkey)).Write(seed, nSeedLen).Finalize(vout.data());

    -   然后，调用私钥的 `Set` 方法，用 512 位的字符串的左边 256 位来初始化私钥的 `keydata` 属性，并且设置私钥的有效标志 `fValid` 属性为真，压缩标志 `fCompressed` 为参数指定的值，默认为真。

            key.Set(vout.data(), vout.data() + 32, true);

    -   调用 `memcpy` 方法，把 512 位的字符串的右边 256 位保存为链码。

            memcpy(chaincode.begin(), vout.data() + 32, 32);

    -   设置主私钥的 `nDepth`、`nChild` 为 0。

            nDepth = 0;
            nChild = 0;

    -   把 `vchFingerprint` 重置为 0。

            memset(vchFingerprint, 0, sizeof(vchFingerprint));

    至此，主私钥终于可用了。

4.  调用主私钥（扩展私钥）的 `Derive` 方法，开始衍生子密钥 `accountKey`。

        masterKey.Derive(accountKey, BIP32_HARDENED_KEY_LIMIT);

    注意，在 bip32 之后，采用硬化衍生子密钥。

    我们来看下 `Derive` 这个方法的执行流程：

    -   设置扩展私钥 `out` 参数的 `nDepth` 为 `nDepth` 加 1。主私钥的 `nDepth` 为 0，参见上面所讲。

    -   获取主公钥的 CKeyID。

             CKeyID id = key.GetPubKey().GetID();

    -   设置置扩展私钥 `out` 参数的 `nChild` 为参数 `_nChild` 的值，这里为 `0x80000000`，10进制为 2147483648。

    -   调用根私钥的 `Derive` 方法，开始衍生私钥。

            return key.Derive(out.key, out.chaincode, _nChild, chaincode);

        这个方法内部执行流程如下：

        -   生成一个向量。

                std::vector<unsigned char, secure_allocator<unsigned char>> vout(64);

        -   把变量 `nChild` 向右移动 31位，如果所得等于 0，那么调用 `GetPubKey` 方法，获得私钥对应的公钥，然后调用 `BIP32Hash` 方法，填充变量 `vout`。在 `BIP32Hash` 方法中使用 `CHMAC_SHA512` 方法，根据链码参数和子密钥数量来填充 `vout` 变量。如果右移所得不等于0，同样调用 `BIP32Hash` 方法，填充变量 `vout`。

                if ((nChild >> 31) == 0) {
                    CPubKey pubkey = GetPubKey();
                    assert(pubkey.size() == CPubKey::COMPRESSED_PUBLIC_KEY_SIZE);
                    BIP32Hash(cc, nChild, *pubkey.begin(), pubkey.begin()+1, vout.data());
                } else {
                    assert(size() == 32);
                    BIP32Hash(cc, nChild, 0, begin(), vout.data());
                }

        -   把变量 `vout` 的内容拷贝到子链码中。

                memcpy(ccChild.begin(), vout.data()+32, 32);

        -   把私钥的内容拷贝到子私钥中。

                memcpy((unsigned char*)keyChild.begin(), begin(), 32);

        -   使用椭圆曲线算法真正初始化子私钥。

                bool ret = secp256k1_ec_privkey_tweak_add(secp256k1_context_sign, (unsigned char*)keyChild.begin(), vout.data());

        -   设置子私钥为压缩的，是否是有效的，并返回。

                keyChild.fCompressed = true;
                keyChild.fValid = ret;
                return ret;

5.  `accountKey` 私钥（扩展私钥）的 `Derive` 方法，开始衍生子密钥 `chainChildKey`。方法前面刚讲过，此处略过。

        accountKey.Derive(chainChildKey, BIP32_HARDENED_KEY_LIMIT+(internal ? 1 : 0));

6.  接下来衍生子私钥 `childKey`，与上面大致相同，可自行阅读。

        do {
            if (internal) {
                chainChildKey.Derive(childKey, hdChain.nInternalChainCounter | BIP32_HARDENED_KEY_LIMIT);
                metadata.hdKeypath = "m/0'/1'/" + std::to_string(hdChain.nInternalChainCounter) + "'";
                hdChain.nInternalChainCounter++;
            }
            else {
                chainChildKey.Derive(childKey, hdChain.nExternalChainCounter | BIP32_HARDENED_KEY_LIMIT);
                metadata.hdKeypath = "m/0'/0'/" + std::to_string(hdChain.nExternalChainCounter) + "'";
                hdChain.nExternalChainCounter++;
            }
        } while (HaveKey(childKey.key.GetPubKey().GetID()));  

7.  设置变量 `secret` 的值为子私钥的 `childKey`。

        secret = childKey.key;

8.  设置变量 `metadata` 的 HD 种子为当前 HD 链的的种子。

        metadata.hd_seed_id = hdChain.seed_id;

9.  更新 HD 链到数据库中。

        if (!batch.WriteHDChain(hdChain))
            throw std::runtime_error(std::string(__func__) + ": Writing HD chain model failed");


### 2.3、WalletBatch::LoadWallet

本方法，从数据库中加载钱包的相关数据。方法的执行逻辑如下：

1.  首先，初始化几个变量。

        CWalletScanState wss;
        bool fNoncriticalErrors = false;
        DBErrors result = DBErrors::LOAD_OK;

2.  从数据库中读取钱包的最小版本。如果最小版本大于当前的最新版本，则返回数据库异常；否则，调用钱包对象的 `LoadMinVersion` 方法，设置钱包的版本相关属性，`nWalletVersion` 属性为数据库读取取的 `nMinVersion`，`nWalletMaxVersion` 属性为 `nWalletMaxVersion` 与这个值的较大者。

        int nMinVersion = 0;
        if (m_batch.Read((std::string)"minversion", nMinVersion))
        {
            if (nMinVersion > FEATURE_LATEST)
                return DBErrors::TOO_NEW;
            pwallet->LoadMinVersion(nMinVersion);
        }

3.  获取数据库游标，如果出错，则返回数据库游标错误。

        Dbc* pcursor = m_batch.GetCursor();
        if (!pcursor)
        {
            pwallet->WalletLogPrintf("Error getting wallet database cursor\n");
            return DBErrors::CORRUPT;
        }

4.  进放 ` while (true){ ... }` 循环读取数据库中的所有数据。具体处理如下：

    -   调用 `ReadAtCursor` 方法从游标中读取对应的数据。如果没有读取到数据，则退出循环。如果出现错误，则调用钱包对象的 `WalletLogPrintf` 方法打印，然后返回数据库错误。

            CDataStream ssKey(SER_DISK, CLIENT_VERSION);
            CDataStream ssValue(SER_DISK, CLIENT_VERSION);
            int ret = m_batch.ReadAtCursor(pcursor, ssKey, ssValue);
            if (ret == DB_NOTFOUND)
                break;
            else if (ret != 0)
            {
                pwallet->WalletLogPrintf("Error reading next record from wallet database\n");
                return DBErrors::CORRUPT;
            }

    -   调用 `ReadKeyValue` 方法，读取游标中当前的内容。如果读取出错，则根据错误进行相应的处理，具体不细说。如果读取数据成功，其内部根据读取到的数据进行具体处理如下：

        -   如果当前的的 Key 是 `name`，那么从流中读取对应的地址到变量 `strAddress`中，调用 `DecodeDestination` 方法解码这个地址，然后设置钱包对象 `mapAddressBook` 集合对应的 `CAddressBookData` 对象的 `name` 属性为流中读取到的名字值。

                if (strType == "name")
                {
                    std::string strAddress;
                    ssKey >> strAddress;
                    ssValue >> pwallet->mapAddressBook[DecodeDestination(strAddress)].name;
                }

        -   否则，如果前的 Key 是 `purpose` ，那么从流中读取对应的地址到变量 `strAddress`中，调用 `DecodeDestination` 方法解码这个地址，然后设置钱包对象 `mapAddressBook` 集合对应的 `CAddressBookData` 对象的 `purpose` 属性为流中读取到的名字值。

                else if (strType == "purpose")
                {
                    std::string strAddress;
                    ssKey >> strAddress;
                    ssValue >> pwallet->mapAddressBook[DecodeDestination(strAddress)].purpose;
                }

        -   否则，如果当前的 Key 是 `tx` ，即交易，那么处理如下。

            从流中读取交易哈希和钱包交易对象，然后调用 `CheckTransaction` 方法，检查读取的钱包交易对象，如果出错，则返回假。

                uint256 hash;
                ssKey >> hash;
                CWalletTx wtx(nullptr /* pwallet */, MakeTransactionRef());
                ssValue >> wtx;
                CValidationState state;
                if (!(CheckTransaction(*wtx.tx, state) && (wtx.GetHash() == hash) && state.IsValid()))
                    return false;

            如果钱包交易对象的 `fTimeReceivedIsTxTime` 属性大于等于 31404，且小于等于 31703，因为撤销序列化在 31600 中进行了变更，所以进行下面的序列化处理。

                if (31404 <= wtx.fTimeReceivedIsTxTime && wtx.fTimeReceivedIsTxTime <= 31703)
                {
                    if (!ssValue.empty())
                    {
                        char fTmp;
                        char fUnused;
                        std::string unused_string;
                        ssValue >> fTmp >> fUnused >> unused_string;
                        strErr = strprintf("LoadWallet() upgrading tx ver=%d %d %s", wtx.fTimeReceivedIsTxTime, fTmp, hash.ToString());
                        wtx.fTimeReceivedIsTxTime = fTmp;
                    }
                    else
                    {
                        strErr = strprintf("LoadWallet() repairing tx ver=%d %s", wtx.fTimeReceivedIsTxTime, hash.ToString());
                        wtx.fTimeReceivedIsTxTime = 0;
                    }
                    wss.vWalletUpgrade.push_back(hash);
                }

            如果钱包交易对象的 `nOrderPos` 为-1，那么设置钱包扫描状态对象的 `fAnyUnordered` 为真。

                if (wtx.nOrderPos == -1)
                    wss.fAnyUnordered = true;

            调用钱包对象的 `LoadToWallet`，加载数据库中读到的钱包交易对象到钱包中。

        -   否则，如果当前的 Key 是 `watchs` ，那么从流中读取对应的序列化脚本，并读取其对应的值。如果其值等于1，那么调用钱包对象的 `LoadWatchOnly` 方法，加载一个只读的脚本。

                wss.nWatchKeys++;
                CScript script;
                ssKey >> script;
                char fYes;
                ssValue >> fYes;
                if (fYes == '1')
                    pwallet->LoadWatchOnly(script);

        -   否则，如果当前的 Key 是 `key` 或者 `wkey`，那么处理如下。

            从流中读取对应的公钥，如果 Key 是 `key`，那么从流中读取出对应的私钥，否则从流中读取对应的钱包私钥，从中取得对应的私钥。

                CPubKey vchPubKey;
                ssKey >> vchPubKey;
                if (!vchPubKey.IsValid())
                {
                    strErr = "Error reading wallet database: CPubKey corrupt";
                    return false;
                }
                CKey key;
                CPrivKey pkey;
                uint256 hash;
                if (strType == "key")
                {
                    wss.nKeys++;
                    ssValue >> pkey;
                } else {
                    CWalletKey wkey;
                    ssValue >> wkey;
                    pkey = wkey.vchPrivKey;
                }

            从流中读取对应的哈希值。

                ssValue >> hash;

            如果哈希不空，则处理如下：

                if (!hash.IsNull())
                {
                    std::vector<unsigned char> vchKey;
                    vchKey.reserve(vchPubKey.size() + pkey.size());
                    vchKey.insert(vchKey.end(), vchPubKey.begin(), vchPubKey.end());
                    vchKey.insert(vchKey.end(), pkey.begin(), pkey.end());

                    if (Hash(vchKey.begin(), vchKey.end()) != hash)
                    {
                        strErr = "Error reading wallet database: CPubKey/CPrivKey corrupt";
                        return false;
                    }

                    fSkipCheck = true;
                }

            调用 `CKey` 对象的 `Load` 方法加载密钥。

                if (!key.Load(pkey, vchPubKey, fSkipCheck))
                {
                    strErr = "Error reading wallet database: CPrivKey corrupt";
                    return false;
                }

            调用钱包对象的 `LoadKey` 方法，加载密钥。

                if (!pwallet->LoadKey(key, vchPubKey))
                {
                    strErr = "Error reading wallet database: LoadKey failed";
                    return false;
                }

        -   否则，如果当前的 Key 是 `mkey`，则从流中读取对应的主密钥，并保存在钱包 `mapMasterKeys` 对应的位置。

                else if (strType == "mkey")
                {
                    unsigned int nID;
                    ssKey >> nID;
                    CMasterKey kMasterKey;
                    ssValue >> kMasterKey;
                    if(pwallet->mapMasterKeys.count(nID) != 0)
                    {
                        strErr = strprintf("Error reading wallet database: duplicate CMasterKey id %u", nID);
                        return false;
                    }
                    pwallet->mapMasterKeys[nID] = kMasterKey;
                    if (pwallet->nMasterKeyMaxID < nID)
                        pwallet->nMasterKeyMaxID = nID;
                }

        -   否则，如果当前的 Key 是 `ckey`，则从流中读取对应的公钥、私钥，并调用钱包对象的 `LoadCryptedKey` 方法，加载加密的密钥。

                else if (strType == "ckey")
                {
                    CPubKey vchPubKey;
                    ssKey >> vchPubKey;
                    if (!vchPubKey.IsValid())
                    {
                        strErr = "Error reading wallet database: CPubKey corrupt";
                        return false;
                    }
                    std::vector<unsigned char> vchPrivKey;
                    ssValue >> vchPrivKey;
                    wss.nCKeys++;

                    if (!pwallet->LoadCryptedKey(vchPubKey, vchPrivKey))
                    {
                        strErr = "Error reading wallet database: LoadCryptedKey failed";
                        return false;
                    }
                    wss.fIsEncrypted = true;
                }

        -   否则，如果当前的 Key 是 `keymeta`，那么从流中读取对应的公钥及其元数据，调用钱包对象的 `LoadKeyMetadata` 方法，加载密钥元数据。

                else if (strType == "keymeta")
                {
                    CPubKey vchPubKey;
                    ssKey >> vchPubKey;
                    CKeyMetadata keyMeta;
                    ssValue >> keyMeta;
                    wss.nKeyMeta++;
                    pwallet->LoadKeyMetadata(vchPubKey.GetID(), keyMeta);
                }

       -    否则，如果当前的 Key 是 `watchmeta`，那么从流中读取脚本和对应的元数据，并调用钱包对象的 `LoadScriptMetadata` 方法，加载脚本的元数据。

                else if (strType == "watchmeta")
                {
                    CScript script;
                    ssKey >> script;
                    CKeyMetadata keyMeta;
                    ssValue >> keyMeta;
                    wss.nKeyMeta++;
                    pwallet->LoadScriptMetadata(CScriptID(script), keyMeta);
                }

        -   否则，如果当前的 Key 是 `defaultkey`，那么从流中读取对应的公钥。

                else if (strType == "defaultkey")
                {
                    CPubKey vchPubKey;
                    ssValue >> vchPubKey;
                    if (!vchPubKey.IsValid()) {
                        strErr = "Error reading wallet database: Default Key corrupt";
                        return false;
                    }
                }

        -   否则，如果当前的 Key 是 `pool`，那么从流中读取对应的密钥池，并调用钱包对象的 `LoadKeyPool` 方法，加载密钥池。

                else if (strType == "pool")
                {
                    int64_t nIndex;
                    ssKey >> nIndex;
                    CKeyPool keypool;
                    ssValue >> keypool;
                    pwallet->LoadKeyPool(nIndex, keypool);
                }

        -   否则，如果当前的 Key 是 `version`，那么从流中读取对应的版本到钱包扫描状态对象的 `nFileVersion` 属性中。如果这个值等于 10300，那么设置钱包对象的这个属性为 300。

                else if (strType == "version")
                {
                    ssValue >> wss.nFileVersion;
                    if (wss.nFileVersion == 10300)
                        wss.nFileVersion = 300;
                }

        -   否则，如果当前的 Key 是 `cscript`，那么从流中读取对应的脚本，并调用钱包对象的 `LoadCScript` 方法加载脚本。

                else if (strType == "cscript")
                {
                    uint160 hash;
                    ssKey >> hash;
                    CScript script;
                    ssValue >> script;
                    if (!pwallet->LoadCScript(script))
                    {
                        strErr = "Error reading wallet database: LoadCScript failed";
                        return false;
                    }
                }

        -   否则，如果当前的 Key 是 `orderposnext`，那么从流中读取对应的值到钱包对象的 `nOrderPosNext` 属性中。

                else if (strType == "orderposnext")
                {
                    ssValue >> pwallet->nOrderPosNext;
                }

        -   否则，如果当前的 Key 是 `destdata`，那么从流中读取对应的数据，并调用钱包对象的 `LoadDestData` 方法，处理这些数据。

                else if (strType == "destdata")
                {
                    std::string strAddress, strKey, strValue;
                    ssKey >> strAddress;
                    ssKey >> strKey;
                    ssValue >> strValue;
                    pwallet->LoadDestData(DecodeDestination(strAddress), strKey, strValue);
                }

        -   否则，如果当前的 Key 是 `hdchain`，那么从流中读取 HD 链，并调用钱包对象的 `SetHDChain`方法，设置钱包的 HD 链中。

                else if (strType == "hdchain")
                {
                    CHDChain chain;
                    ssValue >> chain;
                    pwallet->SetHDChain(chain, true);
                }

        -   否则，如果当前的 Key 是 ``，那么调用钱包的 `` 方法，设置其标志。

                else if (strType == "flags") {
                    uint64_t flags;
                    ssValue >> flags;
                    if (!pwallet->SetWalletFlags(flags, true)) {
                        strErr = "Error reading wallet database: Unknown non-tolerable wallet flags found";
                        return false;
                    }
                }

        -   否则，如果当前的 Key 不是 `bestblock`、`bestblock_nomerkle`、`minversion`、`acentry` ，则钱包扫描状态对象的 `m_unknown_records` 未知道记录属性加1。

5.  返回真。

