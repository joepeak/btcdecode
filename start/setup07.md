#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


### 第7步，加载区块链（`src/init.cpp::AppInitMain()`）

首先，计算缓存的大小。包括：区块索引数据库、区块状态数据库、内存中 UTXO 集。代码如下：

    fReindex = gArgs.GetBoolArg("-reindex", false);
    bool fReindexChainState = gArgs.GetBoolArg("-reindex-chainstate", false);

    // cache size calculations
    int64_t nTotalCache = (gArgs.GetArg("-dbcache", nDefaultDbCache) << 20);
    nTotalCache = std::max(nTotalCache, nMinDbCache << 20); // total cache cannot be less than nMinDbCache
    nTotalCache = std::min(nTotalCache, nMaxDbCache << 20); // total cache cannot be greater than nMaxDbcache
    int64_t nBlockTreeDBCache = std::min(nTotalCache / 8, nMaxBlockDBCache << 20);
    nTotalCache -= nBlockTreeDBCache;
    int64_t nTxIndexCache = std::min(nTotalCache / 8, gArgs.GetBoolArg("-txindex", DEFAULT_TXINDEX) ? nMaxTxIndexCache << 20 : 0);
    nTotalCache -= nTxIndexCache;
    int64_t nCoinDBCache = std::min(nTotalCache / 2, (nTotalCache / 4) + (1 << 23)); // use 25%-50% of the remainder for disk cache
    nCoinDBCache = std::min(nCoinDBCache, nMaxCoinsDBCache << 20); // cap total coins db cache
    nTotalCache -= nCoinDBCache;
    nCoinCacheUsage = nTotalCache; // the rest goes to in-memory cache
    int64_t nMempoolSizeMax = gArgs.GetArg("-maxmempool", DEFAULT_MAX_MEMPOOL_SIZE) * 1000000;
    LogPrintf("Cache configuration:\n");
    LogPrintf("* Using %.1fMiB for block index database\n", nBlockTreeDBCache * (1.0 / 1024 / 1024));
    if (gArgs.GetBoolArg("-txindex", DEFAULT_TXINDEX)) {
        LogPrintf("* Using %.1fMiB for transaction index database\n", nTxIndexCache * (1.0 / 1024 / 1024));
    }

上面代码中首先计算总的缓存大小，其用 nTotalCache 表示，通过-dbcache参数设置，然后这个值要取在 nMinDbCache 和 nMaxDbCache 之间。接下来计算 nBlockTreeDBCache 和 nCoinDBCache 以及 nCoinCacheUsage，并且 `nTotalCache = nBlockTreeDBCache +nCoinDBCache + nCoinCacheUsage` 。

然后，只要加载标志为真且没有收到关闭系统的请求，即进行以下 while 循环。

 -   调用 `UnloadBlockIndex` 方法，卸载区块相关的索引。

    因为循环可能执行多次，所以在每次循环开始都要调用本方法来清除相关的一些变量。

    主要设置包括：活跃区块链的栈顶指针为空、pindexBestInvalid 为空、pindexBestHeader 为空、清空交易池、mapBlocksUnlinked（键为缺少交易的区块索引，值为有对应交易的区块索引）集合为空、vinfoBlockFile 集合为空、最后区块文件句柄 nLastBlockFile 为0、脏区块 setDirtyBlockIndex 集合为空、脏区块文件（句柄）setDirtyFileInfo 集合为空、versionbitscache 缓存清空、清空区块索引集合 mapBlockIndex，等等。

-   重置指向活跃 CCoinsView 的全局智能指针变量 pcoinsTip。

-   重置指向 coins 数据库的全局智能指针变量 pcoinsdbview。

-   重置 CCoinsViewErrorCatcher 的智能指针静态变量 pcoinscatcher。

-   重置指向活跃区块树的全局智能指针变量 pblocktree，并生成新的对象。

    这个对象会读写 `/blocks/index/*` 下面的区块文件。

-   如果 `-reset` 参数为真，那么：

    调用 pblocktree 的 `WriteReindexing` 方法，向数据库中写入数据。并且进一步，如果当前处于修剪模式，调用 `CleanupBlockRevFiles` 方法，清除特定区块的数据文件。

        if (fReset) {
            pblocktree->WriteReindexing(true);
            //If we're reindexing in prune mode, wipe away unusable block files and all undo data files
            if (fPruneMode)
                CleanupBlockRevFiles();
        }

    `WriteReindexing` 方法根据参数值是否为真调用不同方法进行写入。如果参数为真，那么调用 `Write` 方法写入数据；否则调用 `Erase` 方法清除数据。

    `CleanupBlockRevFiles` 方法，首先获取区块的目录，然后用迭代器遍历区块目录。如果当前区块目录是一个常规文件，并且文件名字长度为12，且后缀为 `.dat`，那么进行处理：如果当前文件为区块文件，即 `blk?????.dat` 文件，则放入 mapBlockFiles 集合中，否则，即 `rev?????.dat` 文件，调用 `remove` 方法进行删除。

    遍历 mapBlockFiles 集合，如果当前区块文件的名字不等于变量 nContigCounter 的值，即区块文件名字出现了不连续，那么就删除指定的区块文件。看代码可能更清楚：

        int nContigCounter = 0;
        for (const std::pair<const std::string, fs::path>& item : mapBlockFiles) {
            if (atoi(item.first) == nContigCounter) {
                nContigCounter++;
                continue;
            }
            remove(item.second);
        }

-   如果收到结束请求，则退出循环。

-   调用 `LoadBlockIndex` 方法，加载区块索引。

        if (!LoadBlockIndex(chainparams)) {
            strLoadError = _("Error loading block database");
            break;
        }

    `LoadBlockIndex` 方法里，如果没有设置 `fReindex`，即不重建区块索引，那么调用 `LoadBlockIndexDB` 方法，从数据库文件加载区块索引，即从 `/blocks/index/*` 文件中加载区块。如果加载成功，那么设置区块索引集合 `mapBlockIndex` 为空。如果设置了 `fReindex` 变量，因为以后会进行重建索引，所以这里没有必要先加载索引了。

        bool LoadBlockIndex(const CChainParams& chainparams)
        {
            // Load block index from databases
            bool needs_init = fReindex;
            if (!fReindex) {
                bool ret = LoadBlockIndexDB(chainparams);
                if (!ret) return false;
                needs_init = mapBlockIndex.empty();
            }

            if (needs_init) {
                // Everything here is for *new* reindex/DBs. Thus, though
                // LoadBlockIndexDB may have set fReindex if we shut down
                // mid-reindex previously, we don't check fReindex and
                // instead only check it prior to LoadBlockIndexDB to set
                // needs_init.

                LogPrintf("Initializing databases...\n");
            }
            return true;
        }

-   如果区块索引成功加载，则检查是否包含创世区块。

        if (!mapBlockIndex.empty() && !LookupBlockIndex(chainparams.GetConsensus().hashGenesisBlock)) {
            return InitError(_("Incorrect or no genesis block found. Wrong datadir for network?"));
        }

-   如果某些区块被修剪过（即用户手动删除过某些区块），但又没有处于修剪模式，则退出循环。

        if (fHavePruned && !fPruneMode) {
            strLoadError = _("You need to rebuild the database using -reindex to go back to unpruned mode.  This will redownload the entire blockchain");
            break;
        }

-   如果不重建索引，调用 `LoadGenesisBlock` 加载创世区块。如果失败，则退出循环。

        if (!fReindex && !LoadGenesisBlock(chainparams)) {
            strLoadError = _("Error initializing block database");
            break;
        }

    `LoadGenesisBlock` 方法内部调用 `CChainState::LoadGenesisBlock` 方法进行处理。后者处理逻辑如下：

    -   首先，检查区块索引集合 `mapBlockIndex` 集合中是否包含创世区块。如果包含则支持返回。

    -   然后，调用区块链参数的 `GenesisBlock` 方法，返回其保存的创世区块，并调用强制转化为区块对象。

    -   再然后，调用 `SaveBlockToDisk` 方法，把创世区块保存到硬盘上。

    -   再然后，调用 `AddToBlockIndex` 方法，返回或生成并返回一个区块索引对象 CBlockIndex。

    -   最后，调用 `ReceivedBlockTransactions` 方法，更新区块的相关信息到区块索引对象上。同时使用深度优先搜索方法寻找当前链的所有可能的下一个区块。

    下面我们看下 `ReceivedBlockTransactions` 方法：

    -   设置区块索引对象的交易数量为区块的交易集合的大小。

    -   设置区块索引对象的 nChainTx 为0。

        nChainTx 表示从创世区块到当前区块总共有多少个交易，如果这个值为不等于0，那么说明所有父亲都是有效的。

    -   设置区块索引对象的 nFile ，表示区块所在的具体区块文件。

    -   设置区块索引对象的 nDataPos，表示区块在区块文件中偏移的位置。

    -   设置区块索引对象的 nUndoPos，表示区块在 rev?????.dat 文件中偏移的位置。

    -   设置区块索引对象的 nStatus，表示区块完全可用。

    -   调用 `IsWitnessEnabled` 方法，检查是否开启了隔离见证。如果是，则设置区块索引对象的 nStatus，表示区块支持隔离见证。

    -   调用区块索引对象的 `RaiseValidity` 方法，提高此区块索引的有效性级别。

    -   保存区块索引对象到 setDirtyBlockIndex 集合中。

    -   如果是创世区块或当前区块的父区块已经在链上，那么进行下面的处理。

        生成一个指向区块索引对象指针的队列，然后把当前区块索引放入队列尾部。如果队列不为空，则循环队列处理每一个元素。取得队列中的第一个元素，并从队列中删除。设置当前区块索引对象的 nChainTx 为前一个父区块索引对象的 nChainTx 加上当前区块的交易数量（如果是创世区块，那么父区块索引对象的 nChainTx 为0）；设置区块索引对象的序列号；如果活跃区块链的顶端区块不空或当前区块索引对象在活跃区块链顶端区块之后，则把当前区块索引对象加入到 setBlockIndexCandidates 集合中；不断的从孤立的区块集合 mapBlocksUnlinked 中查找，并将当前链的所有可能的下一个区块保存到 setBlockIndexCandidates 中。
        
            void CChainState::ReceivedBlockTransactions(const CBlock& block, CBlockIndex* pindexNew, const CDiskBlockPos& pos, const Consensus::Params& consensusParams)
            {
                pindexNew->nTx = block.vtx.size();
                pindexNew->nChainTx = 0;
                pindexNew->nFile = pos.nFile;
                pindexNew->nDataPos = pos.nPos;
                pindexNew->nUndoPos = 0;
                pindexNew->nStatus |= BLOCK_HAVE_DATA;
                if (IsWitnessEnabled(pindexNew->pprev, consensusParams)) {
                    pindexNew->nStatus |= BLOCK_OPT_WITNESS;
                }
                pindexNew->RaiseValidity(BLOCK_VALID_TRANSACTIONS);
                setDirtyBlockIndex.insert(pindexNew);

                if (pindexNew->pprev == nullptr || pindexNew->pprev->nChainTx) {
                    // If pindexNew is the genesis block or all parents are BLOCK_VALID_TRANSACTIONS.
                    std::deque<CBlockIndex*> queue;
                    queue.push_back(pindexNew);

                    // Recursively process any descendant blocks that now may be eligible to be connected.
                    while (!queue.empty()) {
                        CBlockIndex *pindex = queue.front();
                        queue.pop_front();
                        pindex->nChainTx = (pindex->pprev ? pindex->pprev->nChainTx : 0) + pindex->nTx;
                        {
                            LOCK(cs_nBlockSequenceId);
                            pindex->nSequenceId = nBlockSequenceId++;
                        }
                        if (chainActive.Tip() == nullptr || !setBlockIndexCandidates.value_comp()(pindex, chainActive.Tip())) {
                            setBlockIndexCandidates.insert(pindex);
                        }
                        std::pair<std::multimap<CBlockIndex*, CBlockIndex*>::iterator, std::multimap<CBlockIndex*, CBlockIndex*>::iterator> range = mapBlocksUnlinked.equal_range(pindex);
                        while (range.first != range.second) {
                            std::multimap<CBlockIndex*, CBlockIndex*>::iterator it = range.first;
                            queue.push_back(it->second);
                            range.first++;
                            mapBlocksUnlinked.erase(it);
                        }
                    }
                } else {
                    if (pindexNew->pprev && pindexNew->pprev->IsValid(BLOCK_VALID_TREE)) {
                        mapBlocksUnlinked.insert(std::make_pair(pindexNew->pprev, pindexNew));
                    }
                }
            }

-   生成两个智能指针对象。

        pcoinsdbview.reset(new CCoinsViewDB(nCoinDBCache, false, fReset || fReindexChainState));
        pcoinscatcher.reset(new CCoinsViewErrorCatcher(pcoinsdbview.get()));

    程序执行到这里，要么是已经重新建立所有区块索引；要么已经将所有区块的索引加载到 mapBlockIndex 中了。

-   升级数据库格式。

        if (!pcoinsdbview->Upgrade()) {
            strLoadError = _("Error upgrading chainstate database");
            break;
        }

-   接下来处理分叉引起的数据库不一致。

        if (!ReplayBlocks(chainparams, pcoinsdbview.get())) {
            strLoadError = _("Unable to replay blocks. You will need to rebuild the database using -reindex-chainstate.");
                break;
        }

    `ReplayBlocks` 函数内部调用 `CChainState` 的同名方法进行处理。后者处理逻辑如下：

    -   生成 CCoinsViewCache 的缓存对象 cache 来缓存 CCoinsView。

    -   调用 `CCoinsView` 子类 `CCoinsViewDB` 的 `GetHeadBlocks` 方法，从数据库中加载 Key 为 `H` 的区块头部哈希，并保存在集合中。

    -   如果前一部的区块头部哈希集合为空，直接返回真。如果集合长度不等于2，则抛出异常。

    -   如果区块索引集合 mapBlockIndex 不包含区块头部哈希集合中第1个，则抛出异常。否则，保存在 pindexNew 变量中。

    -   如果区块头部哈希集合中第2个的内容不为0，进行如下的检查：如果这个值不在区块索引集合 mapBlockIndex 中，则抛出异常。否则，保存在 pindexOld 变量中，调用 `LastCommonAncestor` 方法，查到 pindexOld 和 pindexNew 这两个区块的最后一个共同祖先，并保存为 pindexFork。

    -   然后，开始回滚到旧分支。只要 pindexOld 不等于最后一个共同祖先，就进行循环处理。具体逻辑如下：首先，如果当前区块索引对象的高度大于0，即不是创世区块，那么调用 `ReadBlockFromDisk` 方法，从硬盘加载指定的区块，然后调用 `DisconnectBlock` 方法，断开这个区块连接；其次，保存当前区块索引对象的前父对象为当前索引对象。

    -   最后，从最后一个共同祖先向前滚动到区块链的顶端。

    以上处理说起来可能不是太清楚，下面是具体代码，可以参考代码进行理解。

        bool CChainState::ReplayBlocks(const CChainParams& params, CCoinsView* view)
        {
            LOCK(cs_main);

            CCoinsViewCache cache(view);

            std::vector<uint256> hashHeads = view->GetHeadBlocks();
            if (hashHeads.empty()) return true; // We're already in a consistent state.
            if (hashHeads.size() != 2) return error("ReplayBlocks(): unknown inconsistent state");

            uiInterface.ShowProgress(_("Replaying blocks..."), 0, false);
            LogPrintf("Replaying blocks\n");

            const CBlockIndex* pindexOld = nullptr;  // Old tip during the interrupted flush.
            const CBlockIndex* pindexNew;            // New tip during the interrupted flush.
            const CBlockIndex* pindexFork = nullptr; // Latest block common to both the old and the new tip.

            if (mapBlockIndex.count(hashHeads[0]) == 0) {
                return error("ReplayBlocks(): reorganization to unknown block requested");
            }
            pindexNew = mapBlockIndex[hashHeads[0]];

            if (!hashHeads[1].IsNull()) { // The old tip is allowed to be 0, indicating it's the first flush.
                if (mapBlockIndex.count(hashHeads[1]) == 0) {
                    return error("ReplayBlocks(): reorganization from unknown block requested");
                }
                pindexOld = mapBlockIndex[hashHeads[1]];
                pindexFork = LastCommonAncestor(pindexOld, pindexNew);
                assert(pindexFork != nullptr);
            }

            // Rollback along the old branch.
            while (pindexOld != pindexFork) {
                if (pindexOld->nHeight > 0) { // Never disconnect the genesis block.
                    CBlock block;
                    if (!ReadBlockFromDisk(block, pindexOld, params.GetConsensus())) {
                        return error("RollbackBlock(): ReadBlockFromDisk() failed at %d, hash=%s", pindexOld->nHeight, pindexOld->GetBlockHash().ToString());
                    }
                    LogPrintf("Rolling back %s (%i)\n", pindexOld->GetBlockHash().ToString(), pindexOld->nHeight);
                    DisconnectResult res = DisconnectBlock(block, pindexOld, cache);
                    if (res == DISCONNECT_FAILED) {
                        return error("RollbackBlock(): DisconnectBlock failed at %d, hash=%s", pindexOld->nHeight, pindexOld->GetBlockHash().ToString());
                    }
                    // If DISCONNECT_UNCLEAN is returned, it means a non-existing UTXO was deleted, or an existing UTXO was
                    // overwritten. It corresponds to cases where the block-to-be-disconnect never had all its operations
                    // applied to the UTXO set. However, as both writing a UTXO and deleting a UTXO are idempotent operations,
                    // the result is still a version of the UTXO set with the effects of that block undone.
                }
                pindexOld = pindexOld->pprev;
            }

            // Roll forward from the forking point to the new tip.
            int nForkHeight = pindexFork ? pindexFork->nHeight : 0;
            for (int nHeight = nForkHeight + 1; nHeight <= pindexNew->nHeight; ++nHeight) {
                const CBlockIndex* pindex = pindexNew->GetAncestor(nHeight);
                LogPrintf("Rolling forward %s (%i)\n", pindex->GetBlockHash().ToString(), nHeight);
                if (!RollforwardBlock(pindex, cache, params)) return false;
            }

            cache.SetBestBlock(pindexNew->GetBlockHash());
            cache.Flush();
            uiInterface.ShowProgress("", 100, false);
            return true;
        }

-   当系统走到这一步时，硬盘上的 coinsdb 数据库已经片于一致状态了。现在可以创建指向活跃 CCoinsView 的全局智能指针变量 pcoinsTip。

        pcoinsTip.reset(new CCoinsViewCache(pcoinscatcher.get()));

-   如果 coins 视图不空，那么加载指向最佳区块链的栈顶区块，即最高区块。

        bool is_coinsview_empty = fReset || fReindexChainState || pcoinsTip->GetBestBlock().IsNull();
        if (!is_coinsview_empty) {
            // LoadChainTip sets chainActive based on pcoinsTip's best block
            if (!LoadChainTip(chainparams)) {
                strLoadError = _("Error initializing block database");
                break;
            }
            assert(chainActive.Tip() != nullptr);
        }

    在 `LoadChainTip` 方法中，首先检查活跃区块链的顶端区块，如果存在且其哈希等于 pcoinsTip 指向的最佳区块，即最佳区块链/活跃区块链的顶端区块存在，直接返回真。否则，调用活跃 coins 视图 pcoinsTip 的 `GetBestBlock` 方法返回最佳区块，如果最佳区块不为空并且当前区块链只有创世区块（即区块映射集合长度等于1），调用 `ActivateBestChain` 方法，激活最佳区块链。调用 `LookupBlockIndex` 方法，查找最佳区块的索引。如果可以找到，则设置为活跃区块链的顶端指示区块。

    具体代码如下所示：

        bool LoadChainTip(const CChainParams& chainparams)
        {
            AssertLockHeld(cs_main);

            if (chainActive.Tip() && chainActive.Tip()->GetBlockHash() == pcoinsTip->GetBestBlock()) return true;

            if (pcoinsTip->GetBestBlock().IsNull() && mapBlockIndex.size() == 1) {
                // In case we just added the genesis block, connect it now, so
                // that we always have a chainActive.Tip() when we return.
                LogPrintf("%s: Connecting genesis block...\n", __func__);
                CValidationState state;
                if (!ActivateBestChain(state, chainparams)) {
                    LogPrintf("%s: failed to activate chain (%s)\n", __func__, FormatStateMessage(state));
                    return false;
                }
            }

            // Load pointer to end of best chain
            CBlockIndex* pindex = LookupBlockIndex(pcoinsTip->GetBestBlock());
            if (!pindex) {
                return false;
            }
            chainActive.SetTip(pindex);

            g_chainstate.PruneBlockIndexCandidates();

            LogPrintf("Loaded best chain: hashBestChain=%s height=%d date=%s progress=%f\n",
                chainActive.Tip()->GetBlockHash().ToString(), chainActive.Height(),
                FormatISO8601DateTime(chainActive.Tip()->GetBlockTime()),
                GuessVerificationProgress(chainparams.TxData(), chainActive.Tip()));
            return true;
        }

-   接下来处理数据缺失的情况。

        if (!fReset) {
            uiInterface.InitMessage(_("Rewinding blocks..."));
            if (!RewindBlockIndex(chainparams)) {
                strLoadError = _("Unable to rewind the database to a pre-fork state. You will need to redownload the blockchain");
                break;
            }
        }

    `RewindBlockIndex` 方法中，会断开区块连接，并从区块索引对象中删除相关的区块索引。

-   最后，进行区块数据验证。具体处理可以看代码，此处不细说。

