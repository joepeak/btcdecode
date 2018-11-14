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


##  创建钱包方法 CreateWalletFromFile

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

3.  生成两个变量，一个是当前时间，一个是表示第一次运行钱包的标志。

        int64_t nStart = GetTimeMillis();
        bool fFirstRun = true;

4.  根据钱包名称和绝对路径生成钱包对象。

        std::shared_ptr<CWallet> walletInstance(new CWallet(name, WalletDatabase::Create(path)), ReleaseWallet);

4.  调用钱包对象的 `LoadWallet` 方法加载钱包。如果出现错误，则进行相应的处理。

        DBErrors nLoadWalletRet = walletInstance->LoadWallet(fFirstRun);
        if (nLoadWalletRet != DBErrors::LOAD_OK)
        {
            if (nLoadWalletRet == DBErrors::CORRUPT) {
                InitError(strprintf(_("Error loading %s: Wallet corrupted"), walletFile));
                return nullptr;
            }
            else if (nLoadWalletRet == DBErrors::NONCRITICAL_ERROR)
            {
                InitWarning(strprintf(_("Error reading %s! All keys read correctly, but transaction data or address book entries might be missing or incorrect."),
                    walletFile));
            }
            else if (nLoadWalletRet == DBErrors::TOO_NEW) {
                InitError(strprintf(_("Error loading %s: Wallet requires newer version of %s"), walletFile, _(PACKAGE_NAME)));
                return nullptr;
            }
            else if (nLoadWalletRet == DBErrors::NEED_REWRITE)
            {
                InitError(strprintf(_("Wallet needed to be rewritten: restart %s to complete"), _(PACKAGE_NAME)));
                return nullptr;
            }
            else {
                InitError(strprintf(_("Error loading %s"), walletFile));
                return nullptr;
            }
        }

    钱包对象 `LoadWallet` 方法的主体是生成一个可以访问钱包的数据库的对象，从而调用钱包数据库对象的的 `LoadWallet` 方法来加载钱包。从钱包数据库中加载钱包的方法在下面详细说明，此处略过。

        DBErrors CWallet::LoadWallet(bool& fFirstRunRet)
        {
            LOCK2(cs_main, cs_wallet);
            fFirstRunRet = false;
            DBErrors nLoadWalletRet = WalletBatch(*database,"cr+").LoadWallet(this);
            if (nLoadWalletRet == DBErrors::NEED_REWRITE)
            {
                if (database->Rewrite("\x04pool"))
                {
                    setInternalKeyPool.clear();
                    setExternalKeyPool.clear();
                    m_pool_key_to_index.clear();
                }
            }
            {
                LOCK(cs_KeyStore);
                fFirstRunRet = mapKeys.empty() && mapCryptedKeys.empty() && mapWatchKeys.empty() && setWatchOnly.empty() && mapScripts.empty() && !IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS);
            }
            if (nLoadWalletRet != DBErrors::LOAD_OK)
                return nLoadWalletRet;

            return DBErrors::LOAD_OK;
        }

5.  如果启动参数 `-upgradewallet` 为真，或者没有指定启动参数 `-upgradewallet` 为假，但是第一次运行创建这个钱包，那么进行如下处理：

    -   获取启动参数 `-upgradewallet` 的值，默认为0，保存到变量 `nMaxVersion` 中。

            int nMaxVersion = gArgs.GetArg("-upgradewallet", 0);

    -   如果变量 `nMaxVersion` 为0，即没有指定启动参数，那么设置 `nMaxVersion` 等于 `FEATURE_LATEST（目前为 169900）`，即支持 HD 分割，同时调用钱包对象的 `SetMinVersion` 方法，设置钱包最小版本为这个版本。

            if (nMaxVersion == 0) // the -upgradewallet without argument case
            {
                walletInstance->WalletLogPrintf("Performing wallet upgrade to %i\n", FEATURE_LATEST);
                nMaxVersion = FEATURE_LATEST;
                walletInstance->SetMinVersion(FEATURE_LATEST); // permanently upgrade the wallet immediately
            }
            else
                walletInstance->WalletLogPrintf("Allowing wallet upgrade up to %i\n", nMaxVersion);

    -   如果变量 `nMaxVersion` 小于钱包当前的版本，则直接返回空指针。

            int nMaxVersion = gArgs.GetArg("-upgradewallet", 0);
            if (nMaxVersion < walletInstance->GetVersion())
            {
                InitError(_("Cannot downgrade wallet"));
                return nullptr;
            }

    -   调用钱包对象的 `SetMinVersion` 方法，设置钱包最大版本。

            walletInstance->SetMaxVersion(nMaxVersion);

6.  如果没有指定启动参数 `-upgradewallet` ，或者指定了但为假，那么升级到 HD 钱包。具体处理如下：

    -   如果钱包不支持 `FEATURE_HD_SPLIT`，并且钱包的版本大于等于 139900 （`FEATURE_HD_SPLIT`）且小于等于 169900 （`FEATURE_PRE_SPLIT_KEYPOOL`），则返回空指针。

            int max_version = walletInstance->nWalletVersion;
            if (!walletInstance->CanSupportFeature(FEATURE_HD_SPLIT) && max_version >=FEATURE_HD_SPLIT && max_version < FEATURE_PRE_SPLIT_KEYPOOL) {
                return nullptr;
            }

    -   如果钱包支持 `FEATURE_HD`，且当前没有启用 HD，那么调用 `SetMinVersion` 方法，设置最小版本为 `FEATURE_HD`，同时调用 `GenerateNewSeed` 生成新的随机种子，然后调用 `SetHDSeed` 方法来设置随机种子。

            bool hd_upgrade = false;
            bool split_upgrade = false;
            if (walletInstance->CanSupportFeature(FEATURE_HD) && !walletInstance->IsHDEnabled()) {
                walletInstance->SetMinVersion(FEATURE_HD);
                CPubKey masterPubKey = walletInstance->GenerateNewSeed();
                walletInstance->SetHDSeed(masterPubKey);
                hd_upgrade = true;
            }

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

7.  如果是第一次运行，即第一次创建这个钱包，那么：

    -   设置钱包最小版本为 `FEATURE_LATEST`，当前为 `FEATURE_PRE_SPLIT_KEYPOOL`。

            walletInstance->SetMinVersion(FEATURE_LATEST);

    -   如果创建参数指定不能包含私钥，那么设置钱包这个标记；否则，调用 `GenerateNewSeed` 方法，生成新的随机种子，然后调用 `SetHDSeed` 方法，保存随机种子。

            if ((wallet_creation_flags & WALLET_FLAG_DISABLE_PRIVATE_KEYS)) {
                walletInstance->SetWalletFlag(WALLET_FLAG_DISABLE_PRIVATE_KEYS);
            } else {
                CPubKey seed = walletInstance->GenerateNewSeed();
                walletInstance->SetHDSeed(seed);
            }

    -   如果创建参数没有指定不能包含私钥，那么填充密钥池。如果失败，即不能生成初始密钥，则返回空指针。

            if (!walletInstance->IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS) && !walletInstance->TopUpKeyPool()) {
                return nullptr;
            }

    -   刷新到数据库。

            walletInstance->ChainStateFlushed(chainActive.GetLocator());

8.  否则，如果创建参数指定不能包含私钥，那么返回 NULL。

9.  如果指定了启动参数 `-addresstype`，但是解析失败，则返回空指针。

10. 如果指定了启动参数 `-changetype`，但是解析失败，则返回空指针。

11. 如果设置了最小交易费用，则解析并设置钱包的最小交易费用。

        if (gArgs.IsArgSet("-mintxfee")) {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-mintxfee", ""), n) || 0 == n) {
                return nullptr;
            }
            walletInstance->m_min_fee = CFeeRate(n);
        }

12. 根据不同网络取得是否启用回退费用。如果启动参数设置了 `-fallbackfee` 用以在估算费用不足时将使用的费率，那么就解析设置的费率，如果解析错误，则打印错误日志，并返回空指针，如果解析OK，但是大于规定的最大值，则打印警告日志。如果两者都没有问题，则设置钱包的回退费率。

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

13. 如果启动参数设置了 `-discardfee` 用以规定在费用小于多少时舍弃，那么就解析设置的费率，如果解析错误，则打印错误日志，并返回空指针，如果解析OK，但是大于规定的最大值，则打印警告日志。如果两者都没有问题，则设置钱包的丢弃费率。

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

14. 如果启用参数设置了 `-paytxfee` 指定交易费用，那么就解析设置的费率，如果解析错误，则打印错误日志，并返回空指针，如果解析OK，但是大于规定的最大值，则打印警告日志。如果两者都没有问题，则设置钱包的交易费用。如果交易费用小于规定的最小值，则认为交易费用为为，则打印警告日志，并返回空指针。

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

15. 设置交易确认的平均区块数（默认为6）、是否发送未确认的变更、是否启用 full-RBF opt-in。

        walletInstance->m_confirm_target = gArgs.GetArg("-txconfirmtarget", DEFAULT_TX_CONFIRM_TARGET);
        walletInstance->m_spend_zero_conf_change = gArgs.GetBoolArg("-spendzeroconfchange", DEFAULT_SPEND_ZEROCONF_CHANGE);
        walletInstance->m_signal_rbf = gArgs.GetBoolArg("-walletrbf", DEFAULT_WALLET_RBF);

16. 接下来填充密钥池，如果钱包被锁定，则不进行任何操作。

        walletInstance->TopUpKeyPool();

17. 获取区块链的创世区块索引。如果没有指定重新扫描区块链，或指定不扫描，那么进行下面的操作。

    生成一个可以访问钱包数据库的对象，然后从数据库读取最佳区块，即关键字为 `bestblock` 的区块。如果可以找到这个区块，那么调用 `FindForkInGlobalIndex` 方法，返回分叉的区块。

    最佳区块是一个区块定位器，它描述了块链中到另一个节点的位置，这样如果另一个节点没有相同的分支，它就可以找到最近的公共中继。

        CBlockIndex *pindexRescan = chainActive.Genesis();
        if (!gArgs.GetBoolArg("-rescan", false))
        {
            WalletBatch batch(*walletInstance->database);
            CBlockLocator locator;
            if (batch.ReadBestBlock(locator))
                pindexRescan = FindForkInGlobalIndex(chainActive, locator);
        }

    `pindexRescan` 为重新扫描区块链的区块，默认从创世区块开始，下面会进行重新设置。

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

18. 设置钱包最后一个处理的区块为当前区块链顶部的区块。

        walletInstance->m_last_block_processed = chainActive.Tip();

19. 如果当前区块链顶部的区块存在，且不等于前一步中我们找到的 `pindexRescan` 对应的区块，那么进行下面的处理：

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

20. 调用全局方法 `RegisterValidationInterface` 注册钱包绑定方法作为信号处理器。

21.  调用钱包对象的 `SetBroadcastTransactions` 方法，根据启动参数设置是否广播交易。

22. 返回钱包对象。


##  从数据库中读取钱包的 LoadWallet 方法

在这里，我们仔细看下钱包数据库的 `LoadWallet` 方法。这个方法的执行逻辑如下：

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

