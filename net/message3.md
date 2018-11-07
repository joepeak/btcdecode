##  10、IBD 时，区块头部优先节点获取数据的处理

`IBD` 即初始区块下载的缩写。

在 `IBD` 期间，新的对等节点需要下载大量的区块来使本节点的区块链达到最佳高度。当节点设置了优先下载区块头部时，就会优先下载头部，然后再下载区块，而不是首先下载区块。现在新的比特币核心客户端基本上都是设置为优先下载区块头部，我们在本部分讲述这部分内容。

### 10.1、`getheaders` 消息

作为服务器，响应客户端节点发送的请求多个头部消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2123 行。

以 `if (strCommand == NetMsgType::GETHEADERS) {` 为开始，具体如下：

1.  从输入流中取得区块定位器和结束区块的哈希值。

        CBlockLocator locator;
        uint256 hashStop;
        vRecv >> locator >> hashStop;

2.  如果区块定位器的 `vHave` 数量大于规定的最大数量，那么断开节点的连接，并返回真。

        if (locator.vHave.size() > MAX_LOCATOR_SZ) {
            LogPrint(BCLog::NET, "getheaders locator size %lld > %d, disconnect peer=%d\n", locator.vHave.size(), MAX_LOCATOR_SZ, pfrom->GetId());
            pfrom->fDisconnect = true;
            return true;
        }

3.  如果当前正处于 IBD 中，且远程节点又不在白名单中，则直接返回。
        if (IsInitialBlockDownload() && !pfrom->fWhitelisted) {
            LogPrint(BCLog::NET, "Ignoring getheaders from peer=%d because node is in initial block download\n", pfrom->GetId());
            return true;
        }

4.  调用定位器的 `IsNull` 方法，判断 `vHave` 属性集合是否为空。

5.  如果区块定位器 `vHave` 集合为空，则：

    -   调用 `LookupBlockIndex` 方法，返回停止位置的区块。如果停止位置的区块不存在，则直接返回。

            pindex = LookupBlockIndex(hashStop);
            if (!pindex) {
                return true;
            }

        方法内部从 `mapBlockIndex` 集合里查找哈希对应的区块索引对象。

    -   调用 `BlockRequestAllowed` 方法，判断是否允许当前请求。如果不允许，则直接返回。

        这里的判断是为了防止指纹攻击。

6.  否则，即区块定位器 `vHave` 集合不空，则：

    调用 `FindForkInGlobalIndex` 方法，返回请求的区块索引。如果主链上最后一个区块的哈希存在，则调用活跃区块链的 `Next` 方法，请求下一个区块。

    `FindForkInGlobalIndex` 方法内部遍历区块定位器的 `vHave` 集合，根据当前的哈希的，调用 `LookupBlockIndex` 方法，寻找其对应的区块索引对象，如果该索引对象存在且区块链中包含这个索引，则返回这个区块索引，否则如果该索引对象存在且其祖先是栈顶区块对象，那么返回栈顶元素的区块索引对象。如果找不到对应的索引对象，那么返回创世区块的索引对象。

    代码如下：

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

7.  for 循环遍历活跃区块链上当前区块索引之后的区块，处理如下：

    -   获取当前区块索引对象的区块头部，保存在 `vHeaders` 变量中。

    -   如果已经达到最大长度限制 2000，或达到指定的停止位置，退出循环。

    代码如下：

        std::vector<CBlock> vHeaders;
        int nLimit = MAX_HEADERS_RESULTS;
        for (; pindex; pindex = chainActive.Next(pindex))
        {
            vHeaders.push_back(pindex->GetBlockHeader());
            if (--nLimit <= 0 || pindex->GetBlockHash() == hashStop)
                break;
        }

8.  设置节点状态对象的发送给对等节点的最佳区块索引为当前的区块索引对象或区块链的栈顶索引。

        nodestate->pindexBestHeaderSent = pindex ? pindex : chainActive.Tip();

9.  调用 `PushMessage` 消息，发送 `headers` 消息。

        connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::HEADERS, vHeaders));


### 10.2 `headers` 消息

作为客户端，响应服务器节点发送的多个头部消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2673 行。

以 `if (strCommand == NetMsgType::HEADERS && !fImporting && !fReindex)` 为开始标志，具体如下：

1.  调用 `ReadCompactSize` 方法，计算输入流传递的头部数量。如果头部数量大量 2000，则调用 `Misbehaving` 方法，处罚远程对等节点，然后返回。

        std::vector<CBlockHeader> headers;
        unsigned int nCount = ReadCompactSize(vRecv);
        if (nCount > MAX_HEADERS_RESULTS) {
            LOCK(cs_main);
            Misbehaving(pfrom->GetId(), 20, strprintf("headers message size = %u", nCount));
            return false;
        }

3.  从输入流中取出所有的头部消息，保存到 `headers` 变量中。

        headers.resize(nCount);
        for (unsigned int n = 0; n < nCount; n++) {
            vRecv >> headers[n];
            ReadCompactSize(vRecv); // ignore tx count; assume it is 0.
        }

4.  调用 `ProcessHeadersMessage` 方法，处理接收到的头部。

        bool should_punish = !pfrom->fInbound && !pfrom->m_manual_connection;
        return ProcessHeadersMessage(pfrom, connman, headers, chainparams, should_punish);

    下面我们具体看下这个方法的处理。

    -   如果要请求的头部数量为空，则直接退出方法。

            const CNetMsgMaker msgMaker(pfrom->GetSendVersion());
            size_t nCount = headers.size();
            if (nCount == 0) {
                return true;
            }

    -   如果服务器节点发送的第一个头部对象对应用的区块不存在于 `mapBlockIndex` 集合中，且请求头部的数量（即前面服务器节点返回的头部数量）小于 （`MAX_BLOCKS_TO_ANNOUNCE`）8，则进行如下处理：

        -   节点状态对象的未连接的头部数量增加1，即未能连到区块链上面的头部。

        -   调用 `PushMessage` 方法，发送 `getheaders` 消息，请求区块头部。

        -   调用 `UpdateBlockAvailability` 方法，设置区块状态对象的相关属性。

        -   如果节点发送的未连接到区块链上的头部数量达到规定的最大数量 10，则调用 调用 `Misbehaving` 方法，处罚远程对等节点。

        -   然后返回。

        以上逻辑的代码如下：

            if (!LookupBlockIndex(headers[0].hashPrevBlock) && nCount < MAX_BLOCKS_TO_ANNOUNCE) {
                nodestate->nUnconnectingHeaders++;
                connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETHEADERS, chainActive.GetLocator(pindexBestHeader), uint256()));
                UpdateBlockAvailability(pfrom->GetId(), headers.back().GetHash());
                if (nodestate->nUnconnectingHeaders % MAX_UNCONNECTING_HEADERS == 0) {
                    Misbehaving(pfrom->GetId(), 20);
                }
                return true;
            }

    -   遍历所有的收到的头部，对每个头部进行检查。

            uint256 hashLastBlock;
            for (const CBlockHeader& header : headers) {
                if (!hashLastBlock.IsNull() && header.hashPrevBlock != hashLastBlock) {
                    Misbehaving(pfrom->GetId(), 20, "non-continuous headers sequence");
                    return false;
                }
                hashLastBlock = header.GetHash();
            }
            if (!LookupBlockIndex(hashLastBlock)) {
                received_new_header = true;
            }

    -   调用 `ProcessNewBlockHeaders` 方法，检查每个收到的区块头部。

        `ProcessNewBlockHeaders` 方法内部遍历每个收到的区块头部，调用 `validation.cpp` 文件中的 `CChainState` 对象的 `AcceptBlockHeader` 方法，检查是否可以接受当前的区块头部，如果不能接受则返回假。如果所有的区块头部都可以接受，则返回真。

        如果不能接受所有的头部 ，并且区块状态对象也不是 OK 的，那么进行不同的处理：

        -   如果变量 `nDoS` 大于0，调用 `Misbehaving` 方法，对远程节点进行惩罚，可能导致其被禁止发送。

        -   如果参数 `punish_duplicate_invalid` 为真，且第一个无效的头部对应的区块索引存在，那么设置其断开连接。

        以上逻辑的代码如下：

            CValidationState state;
            CBlockHeader first_invalid_header;
            if (!ProcessNewBlockHeaders(headers, state, chainparams, &pindexLast, &first_invalid_header)) {
                int nDoS;
                if (state.IsInvalid(nDoS)) {
                    LOCK(cs_main);
                    if (nDoS > 0) {
                        Misbehaving(pfrom->GetId(), nDoS, "invalid header received");
                    } else {
                        LogPrint(BCLog::NET, "peer=%d: invalid header received\n", pfrom->GetId());
                    }
                    if (punish_duplicate_invalid && LookupBlockIndex(first_invalid_header.GetHash())) {
                        pfrom->fDisconnect = true;
                    }
                    return false;
                }
            }

    -   设置区块状态对象的未连接头部数量为 0。

        CNodeState *nodestate = State(pfrom->GetId());
        nodestate->nUnconnectingHeaders = 0;

    -   调用 `UpdateBlockAvailability` 方法，更新区块状态对象的相关属性。代码比较简单如下：

            static void UpdateBlockAvailability(NodeId nodeid, const uint256 &hash) EXCLUSIVE_LOCKS_REQUIRED(cs_main) {
                CNodeState *state = State(nodeid);
                assert(state != nullptr);
                ProcessBlockAvailability(nodeid);
                const CBlockIndex* pindex = LookupBlockIndex(hash);
                if (pindex && pindex->nChainWork > 0) {
                    if (state->pindexBestKnownBlock == nullptr || pindex->nChainWork >= state->pindexBestKnownBlock->nChainWork) {
                        state->pindexBestKnownBlock = pindex;
                    }
                } else {
                    state->hashLastUnknownBlock = hash;
                }
            }
    
    -   如果收到的区块头部数量达到规定的最大值 2000 （`MAX_HEADERS_RESULTS`），意味着对等节点还有更多的头部可以发送，那么调用 `PushMessage` 方法，发送 `getheaders` 消息，请求更多的区块头部。

    -   调用 `CanDirectFetch` 方法，检测是否可以直接获取区块数据。

        方法内部比较区块链顶部区块的时间，代码如下：

            static bool CanDirectFetch(const Consensus::Params &consensusParams) EXCLUSIVE_LOCKS_REQUIRED(cs_main)
            {
                return chainActive.Tip()->GetBlockTime() > GetAdjustedTime() - consensusParams.nPowTargetSpacing * 20;
            }

    -   如果可以获取数据，并且区块头部是有效的，并且当前区块链对象的工作量小于最后一个区块索引对象的工作量，进行下面的处理。

        通过最后一个区块对象的 `pprev` 指针，前向遍历所有的区块索引对象，把符合要求的区块对象加入 `vToFetch` 向量中。

            std::vector<const CBlockIndex*> vToFetch;
            const CBlockIndex *pindexWalk = pindexLast;
            while (pindexWalk && !chainActive.Contains(pindexWalk) && vToFetch.size() <= MAX_BLOCKS_IN_TRANSIT_PER_PEER) {
                if (!(pindexWalk->nStatus & BLOCK_HAVE_DATA) &&
                        !mapBlocksInFlight.count(pindexWalk->GetBlockHash()) &&
                        (!IsWitnessEnabled(pindexWalk->pprev, chainparams.GetConsensus()) || State(pfrom->GetId())->fHaveWitness)) {
                    vToFetch.push_back(pindexWalk);
                }
                pindexWalk = pindexWalk->pprev;
            }

        如果当前活跃区块链对象不包含 `pindexWalk` ，那么反转 `vToFetch` 集合的顺序，然后进行遍历，根据每一个区块索引对象构造出一个 `CInv` 对象，并加入 `vGetData` 向量中。然后检查 `vGetData` 向量是否为空，如果不为空，则调用 `PushMessage` 方法，发送 `getdata` 消息，请求数据。

            if (!chainActive.Contains(pindexWalk)) {
                LogPrint(BCLog::NET, "Large reorg, won't direct fetch to %s (%d)\n",
                        pindexLast->GetBlockHash().ToString(),
                        pindexLast->nHeight);
            } else {
                std::vector<CInv> vGetData;
                for (const CBlockIndex *pindex : reverse_iterate(vToFetch)) {
                    if (nodestate->nBlocksInFlight >= MAX_BLOCKS_IN_TRANSIT_PER_PEER) {
                        break;
                    }
                    uint32_t nFetchFlags = GetFetchFlags(pfrom);
                    vGetData.push_back(CInv(MSG_BLOCK | nFetchFlags, pindex->GetBlockHash()));
                    MarkBlockAsInFlight(pfrom->GetId(), pindex->GetBlockHash(), pindex);
                }
                if (vGetData.size() > 0) {
                    if (nodestate->fSupportsDesiredCmpctVersion && vGetData.size() == 1 && mapBlocksInFlight.size() == 1 && pindexLast->pprev->IsValid(BLOCK_VALID_CHAIN)) {
                        vGetData[0] = CInv(MSG_CMPCT_BLOCK, vGetData[0].hash);
                    }
                    connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETDATA, vGetData));
                }
            }

    -   如果当前处于 `IBD` 中，且收到的区块头部数量未达到规定的最大值 2000 （`MAX_HEADERS_RESULTS`），即远程节点没有更多的头部可以发送，则进行下面的处理：

            if (IsInitialBlockDownload() && nCount != MAX_HEADERS_RESULTS) {
                if (nodestate->pindexBestKnownBlock && nodestate->pindexBestKnownBlock->nChainWork < nMinimumChainWork) {
                    if (IsOutboundDisconnectionCandidate(pfrom)) {
                        pfrom->fDisconnect = true;
                    }
                }
            }

    -   最后，进行其他一些简单处理。

            if (!pfrom->fDisconnect && IsOutboundDisconnectionCandidate(pfrom) && nodestate->pindexBestKnownBlock != nullptr) {
                if (g_outbound_peers_with_protect_from_disconnect < MAX_OUTBOUND_PEERS_TO_PROTECT_FROM_DISCONNECT && nodestate->pindexBestKnownBlock->nChainWork >= chainActive.Tip()->nChainWork && !nodestate->m_chain_sync.m_protect) {
                    nodestate->m_chain_sync.m_protect = true;
                    ++g_outbound_peers_with_protect_from_disconnect;
                }
            }

    -   返回真。


### 10.3、`getdata` 消息

作为服务器，响应客户端节点发送的请求数据的消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 1985 行。

以 `if (strCommand == NetMsgType::GETDATA) {` 为开始标志，具体如下：

1.  从输入流中取得需要的数据，保存到 `vInv` 变量中。

        std::vector<CInv> vInv;
        vRecv >> vInv;

2.  如果发送的数量超过 50000 （`MAX_INV_SZ`），则调用 `Misbehaving` 方法，对远程节点进行惩罚，可能导致其被禁止发送。

        if (vInv.size() > MAX_INV_SZ)
        {
            LOCK(cs_main);
            Misbehaving(pfrom->GetId(), 20, strprintf("message getdata size() = %u", vInv.size()));
            return false;
        }

3.  保存请求数据的内容到节点的 `vRecvGetData` 队列中。

4.  调用 `ProcessGetData` 方法，处理客户端请求的数据。

    -   遍历 `vRecvGetData` 队列，如果没有到队列尾部，且当前数据的类型是交易、或隔离见证类型的交易，则进行下面的处理：

        如果当前节点暂停发送，即 `fPauseSend` 属性为真，则退出循环。

        获取当前的 `CInv` 对象，如果交易映射 `mapRelay` 集合中有当前的交易，则调用 `PushMessage` 方法，发送 `tx` 消息；否则，如果节点请求过 `mempool`，即 `timeLastMempoolReq` 属性不空，则从交易池中取得对应的交易。如果交易池中有指定的交易，且进入交易池中的时间在请求 `mempool` 的时间之后，那么调用 `PushMessage` 方法，发送 `tx` 消息。

        如果没有找到对应的交易，则把当前未到的 inv 对象放入未找到集合 `vNotFound`中。

        代码如下：

            while (it != pfrom->vRecvGetData.end() && (it->type == MSG_TX || it->type == MSG_WITNESS_TX)) {
                if (interruptMsgProc)
                    return;
                if (pfrom->fPauseSend)
                    break;
                const CInv &inv = *it;
                it++;
                bool push = false;
                auto mi = mapRelay.find(inv.hash);
                int nSendFlags = (inv.type == MSG_TX ? SERIALIZE_TRANSACTION_NO_WITNESS : 0);
                if (mi != mapRelay.end()) {
                    connman->PushMessage(pfrom, msgMaker.Make(nSendFlags, NetMsgType::TX, *mi->second));
                    push = true;
                } else if (pfrom->timeLastMempoolReq) {
                    auto txinfo = mempool.info(inv.hash);
                    if (txinfo.tx && txinfo.nTime <= pfrom->timeLastMempoolReq) {
                        connman->PushMessage(pfrom, msgMaker.Make(nSendFlags, NetMsgType::TX, *txinfo.tx));
                        push = true;
                    }
                }
                if (!push) {
                    vNotFound.push_back(inv);
                }
            }

    -   如果没有到队列尾部，且没有暂停发送，更进一步，当前请求数据的类型是区块、过滤区块、紧凑区块、隔离见证区块，则调用 `ProcessGetBlockData` 方法，进行处理。

            if (it != pfrom->vRecvGetData.end() && !pfrom->fPauseSend) {
                const CInv &inv = *it;
                if (inv.type == MSG_BLOCK || inv.type == MSG_FILTERED_BLOCK || inv.type == MSG_CMPCT_BLOCK || inv.type == MSG_WITNESS_BLOCK) {
                    it++;
                    ProcessGetBlockData(pfrom, chainparams, inv, connman);
                }
            }

        `ProcessGetBlockData` 方法的主体是调用 `PushMessage` 方法，发送区块、过滤区块、紧凑区块、隔离见证区块。

    -   清除 `vRecvGetData` 队列从开始到当前位置之间所有已经处理过的元素。

            pfrom->vRecvGetData.erase(pfrom->vRecvGetData.begin(), it);

    -   如果请求的某些数据没有找到，则发送 `notfound` 消息。

            connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::NOTFOUND, vNotFound));

####    10.3.1、ProcessGetBlockData

本方法的主要作用是处理客户端请求区块数据。实现如下：

1.  初始化相关变量。

        bool send = false;
        std::shared_ptr<const CBlock> a_recent_block;
        std::shared_ptr<const CBlockHeaderAndShortTxIDs> a_recent_compact_block;
        bool fWitnessesPresentInARecentCompactBlock;
        const Consensus::Params& consensusParams = chainparams.GetConsensus();
        a_recent_block = most_recent_block;
        a_recent_compact_block = most_recent_compact_block;
        fWitnessesPresentInARecentCompactBlock = fWitnessesPresentInMostRecentCompactBlock;

2.  查找对应的区块对象。如果存在，且其保存的总的交易数量大于0，但是没有经过验证，那么设置变量 `need_activate_chain` 为真。

        bool need_activate_chain = false;
        {
            LOCK(cs_main);
            const CBlockIndex* pindex = LookupBlockIndex(inv.hash);
            if (pindex) {
                if (pindex->nChainTx && !pindex->IsValid(BLOCK_VALID_SCRIPTS) &&
                        pindex->IsValid(BLOCK_VALID_TREE)) {
                    need_activate_chain = true;
                }
            }
        }

3.  如果变量 `need_activate_chain` 为真，则调用 `ActivateBestChain` 方法，激活区块链。

        if (need_activate_chain) {
            CValidationState state;
            if (!ActivateBestChain(state, Params(), a_recent_block)) {
                LogPrint(BCLog::NET, "failed to activate chain (%s)\n", FormatStateMessage(state));
            }
        }

4.  如果区块索引对象存在，则调用 `BlockRequestAllowed` 方法，确实是否允许发送区块对象给这个节点。

        const CBlockIndex* pindex = LookupBlockIndex(inv.hash);
        if (pindex) {
            send = BlockRequestAllowed(pindex, consensusParams);
            if (!send) {
                LogPrint(BCLog::NET, "%s: ignoring request from peer=%i for old block that isn't in the main chain\n", __func__, pfrom->GetId());
            }
        }

5.  如果允许发送，但是已经达到输出限制，则断开连接，并设置不允许发送。

        if (send && connman->OutboundTargetReached(true) && ( ((pindexBestHeader != nullptr) && (pindexBestHeader->GetBlockTime() - pindex->GetBlockTime() > HISTORICAL_BLOCK_AGE)) || inv.type == MSG_FILTERED_BLOCK) && !pfrom->fWhitelisted)
        {
            LogPrint(BCLog::NET, "historical block serving limit reached, disconnect peer=%d\n", pfrom->GetId());

            //disconnect node
            pfrom->fDisconnect = true;
            send = false;
        }

6.  如果允许发送，但节点不在白名单中，为了避免因为修剪而导致的泄漏，则断开连接，并设置不允许发送。

        if (send && !pfrom->fWhitelisted && (
                (((pfrom->GetLocalServices() & NODE_NETWORK_LIMITED) == NODE_NETWORK_LIMITED) && ((pfrom->GetLocalServices() & NODE_NETWORK) != NODE_NETWORK) && (chainActive.Tip()->nHeight - pindex->nHeight > (int)NODE_NETWORK_LIMITED_MIN_BLOCKS + 2 /* add two blocks buffer extension for possible races */) )
           )) {
            pfrom->fDisconnect = true;
            send = false;
        }

7.  如果允许发送，并且区块数据存在于硬盘中，则进行下面的处理：

    -   生成区块对象，并根据不同情况进行相应处理。

            std::shared_ptr<const CBlock> pblock;
            if (a_recent_block && a_recent_block->GetHash() == pindex->GetBlockHash()) {
                pblock = a_recent_block;
            } else if (inv.type == MSG_WITNESS_BLOCK) {
                std::vector<uint8_t> block_data;
                if (!ReadRawBlockFromDisk(block_data, pindex, chainparams.MessageStart())) {
                    assert(!"cannot load block from disk");
                }
                connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::BLOCK, MakeSpan(block_data)));
            } else {
                std::shared_ptr<CBlock> pblockRead = std::make_shared<CBlock>();
                if (!ReadBlockFromDisk(*pblockRead, pindex, consensusParams))
                    assert(!"cannot load block from disk");
                pblock = pblockRead;
            }

    -   如果区块对象存在，则进行不同的处理。

        -   如果请求的库存对象类型为区块，则调用 `PushMessage` 方法，发送不带隔离见证信息的 `block` 消息。

        -   如果请求的库存对象类型为隔离见证型区块（详见 BIP144），则调用 `PushMessage` 方法，发送带隔离见证信息的 `block` 消息。

        -   如果请求的库存对象类型为过滤型区块（详见 BIP37），并且设置了过滤器，则调用 `PushMessage` 方法，发送 `merkleblock` 消息，然后发送所有的交易 `tx` 消息。

        -   如果请求的库存对象类型为紧凑型区块，则调用 `PushMessage` 方法，发送 `cmpctblock` 消息或者 `block` 消息。

    -   如果请求的库存对象的哈希等于节点的继续发送哈希，则调用 `PushMessage` 方法，发送 `inv` 消息，继续请求节点发送区块对象。


### 10.4    客户端节点对服务器 `getdata` 消息回复的后续处理

在 `getdata` 消息中，服务器根据请求数据的类型，可以发送区块消息、交易消息等。下面我们对这些消息分别进行说明。


####    10.4.1、`block` 消息

作为客户端，响应服务器节点发送的单个区块的消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2698 行。

以 `if (strCommand == NetMsgType::BLOCK && !fImporting && !fReindex)` 为开始标志，具体如下：

1.  从输入流中取得发送的区块对象。

        std::shared_ptr<CBlock> pblock = std::make_shared<CBlock>();
        vRecv >> *pblock;
        bool forceProcessing = false;
        const uint256 hash(pblock->GetHash());

2.  调用 `ProcessNewBlock` 方法，处理新收到的区块。

        bool fNewBlock = false;
        ProcessNewBlock(chainparams, pblock, forceProcessing, &fNewBlock);

    方法内部处理如下：

    -   首先，调用 `CheckBlock` 方法，检查区块是 OK 的。

            CBlockIndex *pindex = nullptr;
            if (fNewBlock) *fNewBlock = false;
            CValidationState state;
            bool ret = CheckBlock(*pblock, state, chainparams.GetConsensus());

    -   其次，如果第第 1 步检查是 OK 的，则调用区块链状态对象的 `AcceptBlock` 方法，保存区块到硬盘上。

            if (ret) {
                ret = g_chainstate.AcceptBlock(pblock, state, chainparams, &pindex, fForceProcessing, nullptr, fNewBlock);
            }
            if (!ret) {
                GetMainSignals().BlockChecked(*pblock, state);
                return error("%s: AcceptBlock FAILED (%s)", __func__, FormatStateMessage(state));
            }

    -   然后，调用 `NotifyHeaderTip` 方法，通知区块链栈顶已改变。

    -   最后，调用区块链状态对象的 `ActivateBestChain` 方法，把区块连接到区块链上。

            CValidationState state;
            if (!g_chainstate.ActivateBestChain(state, chainparams, pblock))
                return error("%s: ActivateBestChain failed (%s)", __func__, FormatStateMessage(state));

        方法主体是一个 while 循环。最主要的是调用 `ActivateBestChainStep` 方法，激活最佳区块链，把新区块连接到主链上。

        **`ActivateBestChainStep` 方法内部调用 `ConnectTip` 方法来把区块链接到主链，内部调用 `ConnectBlock` 方法进行真正的连接。在把区块连接到区块链上之后，调用 `UpdateTip` 方法，更新栈顶区块**。

3.  如果是新的区块，则设置节点对象的 `nLastBlockTime` 属性，否则，从 `mapBlockSource` 集合中移除对应的对象。

        if (fNewBlock) {
            pfrom->nLastBlockTime = GetTime();
        } else {
            LOCK(cs_main);
            mapBlockSource.erase(pblock->GetHash());
        }

4.  返回真。

#####   10.4.1.1、CheckBlock

本方法主要作用是检查区块是不是 OK 的。方法的第4、5 个参数 `fCheckPOW`、`fCheckMerkleRoot` 默认都为真。实现如下：

1.  如果区块已经验证过，直接返回真。

        if (block.fChecked)
            return true;

2.  调用 `CheckBlockHeader` 方法，检查区块头部是否有效，特别是 POW。如果无效，返回假。

        if (!CheckBlockHeader(block, state, consensusParams, fCheckPOW))
            return false;

    `CheckBlockHeader` 方法内部实现如下：

        if (fCheckPOW && !CheckProofOfWork(block.GetHash(), block.nBits, consensusParams))
            return state.DoS(50, false, REJECT_INVALID, "high-hash", false, "proof of work failed");
        return true;

3.  如果需要检查默克尔头部，则：

    -   首先，调用 `BlockMerkleRoot` 方法，生成默克尔根。

    -   然后，比较区块对象的默克尔根与生成的根是否相同。如果不相同，则调用验证状态对象的 `DoS` 方法，设置其相关属性，特别的如果 `mode` 不等于 `MODE_ERROR`，则设置为 `MODE_INVALID`。

    -   最后，如果变量 `mutated` 为真，则调用验证状态对象的 `DoS` 方法，设置其相关属性。

    以上逻辑的代码如下：

        if (fCheckMerkleRoot) {
            bool mutated;
            uint256 hashMerkleRoot2 = BlockMerkleRoot(block, &mutated);
            if (block.hashMerkleRoot != hashMerkleRoot2)
                return state.DoS(100, false, REJECT_INVALID, "bad-txnmrklroot", true, "hashMerkleRoot mismatch");
            if (mutated)
                return state.DoS(100, false, REJECT_INVALID, "bad-txns-duplicate", true, "duplicate transaction");
        }

4.  检查区块对象的交易数大小是否在指定范围内。

        if (block.vtx.empty() || block.vtx.size() * WITNESS_SCALE_FACTOR > MAX_BLOCK_WEIGHT || ::GetSerializeSize(block, PROTOCOL_VERSION | SERIALIZE_TRANSACTION_NO_WITNESS) * WITNESS_SCALE_FACTOR > MAX_BLOCK_WEIGHT)
            return state.DoS(100, false, REJECT_INVALID, "bad-blk-length", false, "size limits failed");

5.  检查第一个交易是 coinbase 交易，其他交易不是。

        if (block.vtx.empty() || !block.vtx[0]->IsCoinBase())
            return state.DoS(100, false, REJECT_INVALID, "bad-cb-missing", false, "first tx is not coinbase");
        for (unsigned int i = 1; i < block.vtx.size(); i++)
            if (block.vtx[i]->IsCoinBase())
                return state.DoS(100, false, REJECT_INVALID, "bad-cb-multiple", false, "more than one coinbase");

6.  检查每一个交易都是有效的。

        for (const auto& tx : block.vtx)
            if (!CheckTransaction(*tx, state, true))
                return state.Invalid(false, state.GetRejectCode(), state.GetRejectReason(),
                                     strprintf("Transaction check failed (tx hash %s) %s", tx->GetHash().ToString(), state.GetDebugMessage()));

7.  检查签名的大小

        unsigned int nSigOps = 0;
        for (const auto& tx : block.vtx)
        {
            nSigOps += GetLegacySigOpCount(*tx);
        }
        if (nSigOps * WITNESS_SCALE_FACTOR > MAX_BLOCK_SIGOPS_COST)
            return state.DoS(100, false, REJECT_INVALID, "bad-blk-sigops", false, "out-of-bounds SigOpCount");

8.  如果参数 `fCheckPOW`、`fCheckMerkleRoot` 都为真，则设置区块对象已经检查完成。

        if (fCheckPOW && fCheckMerkleRoot)
            block.fChecked = true;

9.  返回真。


#####   10.4.1.2、AcceptBlock

这个方法主要作用是把区块对象保存到硬盘上。方法的 `ppindex`、`dbp` 两个参数为空。实现如下：

1.  初始化各个变量。

        const CBlock& block = *pblock;
        if (fNewBlock) *fNewBlock = false;
        AssertLockHeld(cs_main);
        CBlockIndex *pindexDummy = nullptr;
        CBlockIndex *&pindex = ppindex ? *ppindex : pindexDummy;

    因为 `ppindex` 为空，所以 `pindex` 默认为空指针。

2.  调用 `AcceptBlockHeader` 方法，检查区块头部。

        const CBlock& block = *pblock;
        if (fNewBlock) *fNewBlock = false;
        CBlockIndex *pindexDummy = nullptr;
        CBlockIndex *&pindex = ppindex ? *ppindex : pindexDummy;
        if (!AcceptBlockHeader(block, state, chainparams, &pindex))
            return false;

    `AcceptBlockHeader` 方法执行如下：

    -   获取区块对象的哈希值，并初始其他变量

            uint256 hash = block.GetHash();
            BlockMap::iterator miSelf = mapBlockIndex.find(hash);
            CBlockIndex *pindex = nullptr;

    -   检查区块对象的哈希值是否不等于创世区块的哈希值。如果不等于，则进行下面的检查。

        -   如果哈希对应的区块头部已经存在于 `mapBlockIndex` 集合中，则检查验证状态是否为 `BLOCK_FAILED_MASK`。如果是，则调用 `Invalid` 方法，设置相关的状态属性。

                if (miSelf != mapBlockIndex.end()) {
                    pindex = miSelf->second;
                    if (ppindex)
                        *ppindex = pindex;
                    if (pindex->nStatus & BLOCK_FAILED_MASK)
                        return state.Invalid(error("%s: block %s is marked invalid", __func__, hash.ToString()), 0, "duplicate");
                    return true;
                }

        -   接下来，检查区块头部。代码如下：

                if (!CheckBlockHeader(block, state, chainparams.GetConsensus()))
                    return error("%s: Consensus::CheckBlockHeader: %s, %s", __func__, hash.ToString(), FormatStateMessage(state));

            `CheckBlockHeader` 方法，主要检查 PoW。如果检查失败，则调用 `DoS` 方法，设置相关的状态属性。

        -   再接下来，处理前一个区块的相关检查。

                CBlockIndex* pindexPrev = nullptr;
                BlockMap::iterator mi = mapBlockIndex.find(block.hashPrevBlock);
                if (mi == mapBlockIndex.end())
                    return state.DoS(10, error("%s: prev block not found", __func__), 0, "prev-blk-not-found");
                pindexPrev = (*mi).second;
                if (pindexPrev->nStatus & BLOCK_FAILED_MASK)
                    return state.DoS(100, error("%s: prev block invalid", __func__), REJECT_INVALID, "bad-prevblk");
                if (!ContextualCheckBlockHeader(block, state, chainparams, pindexPrev, GetAdjustedTime()))
                    return error("%s: Consensus::ContextualCheckBlockHeader: %s, %s", __func__, hash.ToString(), FormatStateMessage(state));
                if (!pindexPrev->IsValid(BLOCK_VALID_SCRIPTS)) {
                    for (const CBlockIndex* failedit : m_failed_blocks) {
                        if (pindexPrev->GetAncestor(failedit->nHeight) == failedit) {
                            assert(failedit->nStatus & BLOCK_FAILED_VALID);
                            CBlockIndex* invalid_walk = pindexPrev;
                            while (invalid_walk != failedit) {
                                invalid_walk->nStatus |= BLOCK_FAILED_CHILD;
                                setDirtyBlockIndex.insert(invalid_walk);
                                invalid_walk = invalid_walk->pprev;
                            }
                            return state.DoS(100, error("%s: prev block invalid", __func__), REJECT_INVALID, "bad-prevblk");
                        }
                    }
                }
    
    -   如果 `pindex` 为空，则调用 `AddToBlockIndex` 方法，生成一个新的区块索引对象并进行初始化，然后添加到 `mapBlockIndex` 集合中。

    -   接下来，调用 `CheckBlockIndex` 方法，检查所有的区块索引对象。

    -   最后，返回真。

3.  初始化表示区块是不是已经存在的变量。

        bool fAlreadyHave = pindex->nStatus & BLOCK_HAVE_DATA;

4.  通过比较区块索引对象的总工作量与活跃区块链顶部区块索引的总工作量，确定是否还有更多的工作要做。

        bool fHasMoreOrSameWork = (chainActive.Tip() ? pindex->nChainWork >= chainActive.Tip()->nChainWork : true);

5.  检查区块索引的高度是不是远远高于活跃区块链顶部。

        bool fTooFarAhead = (pindex->nHeight > int(chainActive.Height() + MIN_BLOCKS_TO_KEEP));

    `MIN_BLOCKS_TO_KEEP` 表示不能修剪的最小调试，当前是 288。

6.  如果已经拥有这个区块，则直接返回真。

         if (fAlreadyHave) return true;

7.  如果没有请求这个区块，则处理如下：

        if (!fRequested) {  // If we didn't ask for it:
            if (pindex->nTx != 0) return true;    // This is a previously-processed block that was pruned
            if (!fHasMoreOrSameWork) return true; // Don't process less-work chains
            if (fTooFarAhead) return true;        // Block height is too high
            if (pindex->nChainWork < nMinimumChainWork) return true;
        }

8.  调用 `CheckBlock` 、`ContextualCheckBlock` 两个方法进行检查。如果任何一个方法检查不通过，则返回错误。

        if (!CheckBlock(block, state, chainparams.GetConsensus()) ||
            !ContextualCheckBlock(block, state, chainparams.GetConsensus(), pindex->pprev)) {
            if (state.IsInvalid() && !state.CorruptionPossible()) {
                pindex->nStatus |= BLOCK_FAILED_VALID;
                setDirtyBlockIndex.insert(pindex);
            }
            return error("%s: %s", __func__, FormatStateMessage(state));
        }

    其中，`CheckBlock` 方法，前面已经调用过，这里又一次调用。`ContextualCheckBlock` 方法进行上下文检查。

9.  **如果不是在 IBD，且区块链头部与指定区块的父区块相同，则调用 `GetMainSignals().NewPoWValidBlock` 方法，处理新加入的区块**。

        if (!IsInitialBlockDownload() && chainActive.Tip() == pindex->pprev)
            GetMainSignals().NewPoWValidBlock(pindex, pblock);

10.  调用 `SaveBlockToDisk` 方法，把区块保存到硬盘上。调用 `ReceivedBlockTransactions` 方法，标记区及其数据为已经收到和检查。

        if (fNewBlock) *fNewBlock = true;
        try {
            CDiskBlockPos blockPos = SaveBlockToDisk(block, pindex->nHeight, chainparams, dbp);
            if (blockPos.IsNull()) {
                state.Error(strprintf("%s: Failed to find position to write new block to disk", __func__));
                return false;
            }
            ReceivedBlockTransactions(block, pindex, blockPos, chainparams.GetConsensus());
        } catch (const std::runtime_error& e) {
            return AbortNode(state, std::string("System error: ") + e.what());
        }

    `ReceivedBlockTransactions` 方法，主要是修改区块索引对象的相关属性。

11.  调用 `FlushStateToDisk` 方法，刷新状态到硬盘上。

        FlushStateToDisk(chainparams, state, FlushStateMode::NONE);

12.  调用 `CheckBlockIndex` 方法，进行一些检查。

        CheckBlockIndex(chainparams.GetConsensus());

13. 返回真。


#####   10.4.1.3、ContextualCheckBlock

本方法主要进行区块相关的上下文检查。实现如下：

1.  计算区块的高度。

        const int nHeight = pindexPrev == nullptr ? 0 : pindexPrev->nHeight + 1;

2.  强制使用 BIP113 的 MTP。

        int nLockTimeFlags = 0;
        if (VersionBitsState(pindexPrev, consensusParams, Consensus::DEPLOYMENT_CSV, versionbitscache) == ThresholdState::ACTIVE) {
            nLockTimeFlags |= LOCKTIME_MEDIAN_TIME_PAST;
        }
        int64_t nLockTimeCutoff = (nLockTimeFlags & LOCKTIME_MEDIAN_TIME_PAST)
                                  ? pindexPrev->GetMedianTimePast()
                                  : block.GetBlockTime();
3.  检查所有交易是否已经完成。

        for (const auto& tx : block.vtx) {
            if (!IsFinalTx(*tx, nHeight, nLockTimeCutoff)) {
                return state.DoS(10, false, REJECT_INVALID, "bad-txns-nonfinal", false, "non-final transaction");
            }
        }

4.  强制执行 coinbase 以序列化区块高度开始。

        if (nHeight >= consensusParams.BIP34Height)
        {
            CScript expect = CScript() << nHeight;
            if (block.vtx[0]->vin[0].scriptSig.size() < expect.size() ||
                !std::equal(expect.begin(), expect.end(), block.vtx[0]->vin[0].scriptSig.begin())) {
                return state.DoS(100, false, REJECT_INVALID, "bad-cb-height", false, "block height mismatch in coinbase");
            }
        }

5.  进行验证隔离见证相关的验证

        bool fHaveWitness = false;
        if (VersionBitsState(pindexPrev, consensusParams, Consensus::DEPLOYMENT_SEGWIT, versionbitscache) == ThresholdState::ACTIVE) {
            int commitpos = GetWitnessCommitmentIndex(block);
            if (commitpos != -1) {
                bool malleated = false;
                uint256 hashWitness = BlockWitnessMerkleRoot(block, &malleated);
                if (block.vtx[0]->vin[0].scriptWitness.stack.size() != 1 || block.vtx[0]->vin[0].scriptWitness.stack[0].size() != 32) {
                    return state.DoS(100, false, REJECT_INVALID, "bad-witness-nonce-size", true, strprintf("%s : invalid witness reserved value size", __func__));
                }
                CHash256().Write(hashWitness.begin(), 32).Write(&block.vtx[0]->vin[0].scriptWitness.stack[0][0], 32).Finalize(hashWitness.begin());
                if (memcmp(hashWitness.begin(), &block.vtx[0]->vout[commitpos].scriptPubKey[6], 32)) {
                    return state.DoS(100, false, REJECT_INVALID, "bad-witness-merkle-match", true, strprintf("%s : witness merkle commitment mismatch", __func__));
                }
                fHaveWitness = true;
            }
        }
        if (!fHaveWitness) {
          for (const auto& tx : block.vtx) {
                if (tx->HasWitness()) {
                    return state.DoS(100, false, REJECT_INVALID, "unexpected-witness", true, strprintf("%s : unexpected witness data found", __func__));
                }
            }
        }

6.  验证 weight，具体见 BIP 141。

        if (GetBlockWeight(block) > MAX_BLOCK_WEIGHT) {
            return state.DoS(100, false, REJECT_INVALID, "bad-blk-weight", false, strprintf("%s : weight limit failed", __func__));
        }

7.  返回真。


####    10.4.2、`cmpctblock` 消息

作为客户端，响应服务器节点发送的单个区块的消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2377 行。

以 `if (strCommand == NetMsgType::CMPCTBLOCK && !fImporting && !fReindex)` 为开始标志，具体如下：

1.  获取服务器发送的紧凑区块对象。

        CBlockHeaderAndShortTxIDs cmpctblock;
        vRecv >> cmpctblock;
        bool received_new_header = false;

2.  查找前一个区块的索引对象，如果不存在，并且当前没有在 IBD，那么调用 `PushMessage` 方法，发送 `getheaders` 消息。

        if (!LookupBlockIndex(cmpctblock.header.hashPrevBlock)) {
            if (!IsInitialBlockDownload())
                connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETHEADERS, chainActive.GetLocator(pindexBestHeader), uint256()));
            return true;
        }

3.  查找区块自身的索引对象，如果不存在，则设置变量 `received_new_header` 为真。

        if (!LookupBlockIndex(cmpctblock.header.GetHash())) {
            received_new_header = true;
        }

4.  调用 `ProcessNewBlockHeaders` 方法，验证接收到的区块头部对象。如果区块头部对象验证失败，则进行下面的处理。

        const CBlockIndex *pindex = nullptr;
        CValidationState state;
        if (!ProcessNewBlockHeaders({cmpctblock.header}, state, chainparams, &pindex)) {
            int nDoS;
            if (state.IsInvalid(nDoS)) {
                if (nDoS > 0) {
                    LOCK(cs_main);
                    Misbehaving(pfrom->GetId(), nDoS, strprintf("Peer %d sent us invalid header via cmpctblock\n", pfrom->GetId()));
                } else {
                    LogPrint(BCLog::NET, "Peer %d sent us invalid header via cmpctblock\n", pfrom->GetId());
                }
                return true;
            }
        }

    `ProcessNewBlockHeaders` 方法处理过程，前面已经讲解过，此处不讲解。

5.  构造一个输入流。

        bool fProcessBLOCKTXN = false;
        CDataStream blockTxnMsg(SER_NETWORK, PROTOCOL_VERSION);

6.  初始化以下变量。

        bool fRevertToHeaderProcessing = false;
        std::shared_ptr<CBlock> pblock = std::make_shared<CBlock>();
        bool fBlockReconstructed = false;

7.  设置 `UpdateBlockAvailability` 方法，更新区块索引对象的相关属性。

8.  如果获取的是一个新的头部，并且区块索引对象上保存的工作量大于活跃区块链顶部索引对象的工作量，则更新状态对象的 `m_last_block_announcement` 属性。

        if (received_new_header && pindex->nChainWork > chainActive.Tip()->nChainWork) {
            nodestate->m_last_block_announcement = GetTime();
        }

9.  初始化变量 `blockInFlightIt`、`fAlreadyInFlight`。

        std::map<uint256, std::pair<NodeId, std::list<QueuedBlock>::iterator> >::iterator blockInFlightIt = mapBlocksInFlight.find(pindex->GetBlockHash());
        bool fAlreadyInFlight = blockInFlightIt != mapBlocksInFlight.end();

10.  如果索引对象对应的区块已经在硬盘文件中，则直接返回。

        if (pindex->nStatus & BLOCK_HAVE_DATA) // Nothing to do here
            return true;

11. 区块索引对象上保存的工作量小于活跃区块链顶部索引对象的工作量，或者索引对象的交易数量不为0，进一步索引对象对应的区块哈希存在于 `mapBlocksInFlight` 集合中，即我们已经有这个区块，重新请求这个区块。

        if (pindex->nChainWork <= chainActive.Tip()->nChainWork || pindex->nTx != 0) {
            if (fAlreadyInFlight) {
                std::vector<CInv> vInv(1);
                vInv[0] = CInv(MSG_BLOCK | GetFetchFlags(pfrom), cmpctblock.header.GetHash());
                connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETDATA, vInv));
            }
            return true;
        }

12. 如果我们没有请求过这个区块，并且不能直接获取数据，那么直接返回真。

        if (!fAlreadyInFlight && !CanDirectFetch(chainparams.GetConsensus()))
            return true;

13. 如果网络启用了隔离见证，但是状态对象不支持隔离见证，直接返回真。

        if (IsWitnessEnabled(pindex->pprev, chainparams.GetConsensus()) && !nodestate->fSupportsDesiredCmpctVersion) {
            return true;
        }

14. 如果区块索引的高度小于活跃区块链高度加 2，则进行下面的处理：

    -   如果本地没有请求过这个区块，并且状态对象的 `nBlocksInFlight` 属性小于允许从同一个节点重新请求的区块数量，或者本地有请求过这个区块，并且 `blockInFlightIt` 对应的节点是当前节点，那么：

        -   调用 `MarkBlockAsInFlight` 方法，把区块的哈希放在 `mapBlocksInFlight` 集合中。如果方法返回假，即区块的哈希已经存在于集合中，进行如下处理：

                std::list<QueuedBlock>::iterator* queuedBlockIt = nullptr;
                if (!MarkBlockAsInFlight(pfrom->GetId(), pindex->GetBlockHash(), pindex, &queuedBlockIt)) {
                    if (!(*queuedBlockIt)->partialBlock)
                        (*queuedBlockIt)->partialBlock.reset(new PartiallyDownloadedBlock(&mempool));
                    else {
                        LogPrint(BCLog::NET, "Peer sent us compact block we were already syncing!\n");
                        return true;
                    }
                }
        -   调用 `PartiallyDownloadedBlock` 对象的 `InitData` 方法，对紧凑区块进行处理。

                PartiallyDownloadedBlock& partialBlock = *(*queuedBlockIt)->partialBlock;
                ReadStatus status = partialBlock.InitData(cmpctblock, vExtraTxnForCompact);
                if (status == READ_STATUS_INVALID) {
                    MarkBlockAsReceived(pindex->GetBlockHash()); // Reset in-flight state in case of whitelist
                    Misbehaving(pfrom->GetId(), 100, strprintf("Peer %d sent us invalid compact block\n", pfrom->GetId()));
                    return true;
                } else if (status == READ_STATUS_FAILED) {
                   std::vector<CInv> vInv(1);
                    vInv[0] = CInv(MSG_BLOCK | GetFetchFlags(pfrom), cmpctblock.header.GetHash());
                    connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETDATA, vInv));
                    return true;
                }

        -   生成并初始化 `BlockTransactionsRequest` 对象，向远程节点请求区块中包含的交易数据。

                BlockTransactionsRequest req;
                for (size_t i = 0; i < cmpctblock.BlockTxCount(); i++) {
                    if (!partialBlock.IsTxAvailable(i))
                        req.indexes.push_back(i);
                }
                if (req.indexes.empty()) {
                    BlockTransactions txn;
                    txn.blockhash = cmpctblock.header.GetHash();
                    blockTxnMsg << txn;
                    fProcessBLOCKTXN = true;
                } else {
                    req.blockhash = pindex->GetBlockHash();
                    connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETBLOCKTXN, req));
                }

    -   否则，区块或者已经在 flight ，或者对等节点有太多未完成的块可以下载，从而进行下面处理。

            PartiallyDownloadedBlock tempBlock(&mempool);
            ReadStatus status = tempBlock.InitData(cmpctblock, vExtraTxnForCompact);
            if (status != READ_STATUS_OK) {
                return true;
            }
            std::vector<CTransactionRef> dummy;
            status = tempBlock.FillBlock(*pblock, dummy);
            if (status == READ_STATUS_OK) {
                fBlockReconstructed = true;
            }

15. 否则，即区块索引的高度不小于活跃区块链高度加 2，则进行下面的处理：

    如果已经请求过这个区块，但可能与我们目前已有的数据相关太远，所以没用，要重新发起请求，否则，即这个区块是公布的，那么设置 `fRevertToHeaderProcessing` 为真。

        if (fAlreadyInFlight) {
            std::vector<CInv> vInv(1);
            vInv[0] = CInv(MSG_BLOCK | GetFetchFlags(pfrom), cmpctblock.header.GetHash());
            connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETDATA, vInv));
            return true;
        } else {
            fRevertToHeaderProcessing = true;
        }

16. 如果变量 `fProcessBLOCKTXN` 为真，即所有的区块交易都是可用的，那么调用 `ProcessMessage` 方法，处理这些交易。

17. 如果变量 `fRevertToHeaderProcessing` 为真，那么调用 `ProcessHeadersMessage` 方法，处理头部消息。

18. 如果变量 `fBlockReconstructed` 为真，进行下面的处理。

        if (fBlockReconstructed) {
           {
                LOCK(cs_main);
                mapBlockSource.emplace(pblock->GetHash(), std::make_pair(pfrom->GetId(), false));
            }
            bool fNewBlock = false;
           ProcessNewBlock(chainparams, pblock, /*fForceProcessing=*/true, &fNewBlock);
            if (fNewBlock) {
                pfrom->nLastBlockTime = GetTime();
            } else {
                LOCK(cs_main);
                mapBlockSource.erase(pblock->GetHash());
            }
            LOCK(cs_main); // hold cs_main for CBlockIndex::IsValid()
            if (pindex->IsValid(BLOCK_VALID_TRANSACTIONS)) {
                MarkBlockAsReceived(pblock->GetHash());
            }
        }

20. 返回真。


####    10.4.3、`tx` 消息

作为客户端，响应服务器节点发送的交易消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2190 行。

以 `if (strCommand == NetMsgType::TX) {` 为开始标志，具体如下：


1.  如果不允许中继交易，并且节点不在白名单中，则直接返回真。

        if (!fRelayTxes && (!pfrom->fWhitelisted || !gArgs.GetBoolArg("-whitelistrelay", DEFAULT_WHITELISTRELAY)))
        {
            LogPrint(BCLog::NET, "transaction sent in violation of protocol peer=%d\n", pfrom->GetId());
            return true;
        }

2.  从输入流中取得服务器节点发送的交易信息

        std::deque<COutPoint> vWorkQueue;
        std::vector<uint256> vEraseQueue;
        CTransactionRef ptx;
        vRecv >> ptx;
        const CTransaction& tx = *ptx;

3.  构造 `CInv` 对象，并调用节点的 `AddInventoryKnown` 方法，把库存对象加入 `filterInventoryKnown` 集合中。

        CInv inv(MSG_TX, tx.GetHash());
        pfrom->AddInventoryKnown(inv);

4.  从集合的请求集合和 `mapAlreadyAskedFor` 集合中移除已收到的库存的对象。

        bool fMissingInputs = false;
        CValidationState state;
        pfrom->setAskFor.erase(inv.hash);
        mapAlreadyAskedFor.erase(inv.hash);

5.  调用 `AlreadyHave` 确定是否拥有这个库存对象，调用 `AcceptToMemoryPool` 方法，确定是否可以把这个库存对象加入内存池。如果两都满足，则进行如下处理：

    -   调用 `RelayTransaction` 方法，中继收到的交易。

        方法内部生成库存对象，通过调用所有节点的 `PushInventory` 方法，放入每个节点的 `setInventoryTxToSend` 或者 `vInventoryBlockToSend` 集合。

    -   把交易的所有输出放入变量 `vWorkQueue` 集合中。

            for (unsigned int i = 0; i < tx.vout.size(); i++) {
                vWorkQueue.emplace_back(inv.hash, i);
            }

    -   更新节点最后一次收到交易的时间。

            pfrom->nLastTXTime = GetTime();

    -   生成变量 `std::set<NodeId> setMisbehaving;`。

    -   遍历所有的交易输出，进行下面的处理：

        -   从 `mapOrphanTransactionsByPrev` 集合中查找当前输出对应的孤儿交易集合。同时从输出集合中删除当前的输出。如果当前的输出没有对应的孤儿交易，则处理下一个。

                auto itByPrev = mapOrphanTransactionsByPrev.find(vWorkQueue.front());
                vWorkQueue.pop_front();
                if (itByPrev == mapOrphanTransactionsByPrev.end())
                    continue;

        -   遍历当前输出对应的孤儿交易。

            首先，取得当前孤儿交易对应的相关变量。

                const CTransactionRef& porphanTx = (*mi)->second.tx;
                const CTransaction& orphanTx = *porphanTx;
                const uint256& orphanHash = orphanTx.GetHash();
                NodeId fromPeer = (*mi)->second.fromPeer;
                bool fMissingInputs2 = false;
                CValidationState stateDummy;

            然后，判断 `setMisbehaving` 集合中是否包含这个节点ID。如果包含，则处理下一个。

                if (setMisbehaving.count(fromPeer))
                    continue;

            接下来，如果可以把这个孤儿交易放入内存池中，则把这个孤儿交易对应的输出放入 `vWorkQueue` 集合，同时把这个孤儿交易放入删除的孤儿集合 `vEraseQueue` 中。

                if (AcceptToMemoryPool(mempool, stateDummy, porphanTx, &fMissingInputs2, &lRemovedTxn, false /* bypass_limits */, 0 /* nAbsurdFee */)) {
                    LogPrint(BCLog::MEMPOOL, "   accepted orphan tx %s\n", orphanHash.ToString());
                    RelayTransaction(orphanTx, connman);
                    for (unsigned int i = 0; i < orphanTx.vout.size(); i++) {
                        vWorkQueue.emplace_back(orphanHash, i);
                    }
                    vEraseQueue.push_back(orphanHash);
                }

            否则，如果不能把这个孤儿交易放入内存池中，并且变量 `fMissingInputs2` 为假，则进行不同的处理。如果发送的是无效的孤儿交易，则惩罚节点；如果有输入，但不能放入内存池中，可能是非标准交易，则放入删除的孤儿集合 `vEraseQueue` 中。

                else if (!fMissingInputs2)
                {
                    int nDos = 0;
                    if (stateDummy.IsInvalid(nDos) && nDos > 0)
                    {
                        Misbehaving(fromPeer, nDos);
                        setMisbehaving.insert(fromPeer);
                    }
                    vEraseQueue.push_back(orphanHash);
                    if (!orphanTx.HasWitness() && !stateDummy.CorruptionPossible()) {
                        // Do not use rejection cache for witness transactions or
                        // witness-stripped transactions, as they can have been malleated.
                        // See https://github.com/bitcoin/bitcoin/issues/8279 for details.
                        assert(recentRejects);
                        recentRejects->insert(orphanHash);
                    }
                }

    -   遍历所有因为收到这个交易而需要从孤儿交易队列中删除的交易，调用 `EraseOrphanTx` 方法，删除对应的交易。

            for (const uint256& hash : vEraseQueue)
                EraseOrphanTx(hash);

6.  否则，如果缺少输入，则进行如下处理：

    -   检查所有交易的输入引用的输出是否被拒绝。

            bool fRejectedParents = false; // It may be the case that the orphans parents have all been rejected
            for (const CTxIn& txin : tx.vin) {
                if (recentRejects->contains(txin.prevout.hash)) {
                    fRejectedParents = true;
                    break;
                }
            }

    -   如果所有交易的输入引用的输出都没有被拒绝，那么根据这些输入生成对应的库存对象，并加入节点的 `filterInventoryKnown` 集合中，同时，把当前的交易加入孤儿交易集合中。相反，如果有某个交易的输入引用的输出被拒绝，则把交易加入 `recentRejects` 集合中。

            if (!fRejectedParents) {
                uint32_t nFetchFlags = GetFetchFlags(pfrom);
                for (const CTxIn& txin : tx.vin) {
                    CInv _inv(MSG_TX | nFetchFlags, txin.prevout.hash);
                    pfrom->AddInventoryKnown(_inv);
                    if (!AlreadyHave(_inv)) pfrom->AskFor(_inv);
                }
                AddOrphanTx(ptx, pfrom->GetId());
            } else {
                recentRejects->insert(tx.GetHash());
            }

7.  以上两者都不符合，则

    -   根据交易是隔离见证数据 和 witness-stripped 交易，进行不同的处理。

        if (!tx.HasWitness() && !state.CorruptionPossible()) {
            recentRejects->insert(tx.GetHash());
            if (RecursiveDynamicUsage(*ptx) < 100000) {
                AddToCompactExtraTransactions(ptx);
            }
        } else if (tx.HasWitness() && RecursiveDynamicUsage(*ptx) < 100000) {
            AddToCompactExtraTransactions(ptx);
        }

    -   如果节点在白名单中，且指定了白名单强制中继交易，进行下面的处理。

            if (pfrom->fWhitelisted && gArgs.GetBoolArg("-whitelistforcerelay", DEFAULT_WHITELISTFORCERELAY)) {
                int nDoS = 0;
                if (!state.IsInvalid(nDoS) || nDoS == 0) {
                    LogPrintf("Force relaying tx %s from whitelisted peer=%d\n", tx.GetHash().ToString(), pfrom->GetId());
                    RelayTransaction(tx, connman);
                } else {
                    LogPrintf("Not relaying invalid transaction %s from whitelisted peer=%d (%s)\n", tx.GetHash().ToString(), pfrom->GetId(), FormatStateMessage(state));
                }
            }

8.  如果 `lRemovedTxn` 集合不空，则加入 `vExtraTxnForCompact` 集合中。

        for (const CTransactionRef& removedTx : lRemovedTxn)
            AddToCompactExtraTransactions(removedTx);

9.  根据验证结果，进行不同的处理。

        int nDoS = 0;
        if (state.IsInvalid(nDoS))
        {
            if (enable_bip61 && state.GetRejectCode() > 0 && state.GetRejectCode() < REJECT_INTERNAL) {
                connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::REJECT, strCommand, (unsigned char)state.GetRejectCode(),
                                   state.GetRejectReason().substr(0, MAX_REJECT_MESSAGE_LENGTH), inv.hash));
            }
            if (nDoS > 0) {
                Misbehaving(pfrom->GetId(), nDoS);
            }
        }

10. 返回真。


####    10.4.4、`inv` 消息

作为客户端，响应服务器节点发送的库存的消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 1928 行。

以 `if (strCommand == NetMsgType::INV) {` 为开始标志，具体如下：

1.  从输入流中取得库存消息 `vInv`。如果发送库存消息的数量大于 `inv` 消息允许的最大长度 50000，则：调用 `Misbehaving` 方法，对远程节点进行惩罚，可能导致被禁止；然后退出处理。

        std::vector<CInv> vInv;
        vRecv >> vInv;
        if (vInv.size() > MAX_INV_SZ)
        {
            LOCK(cs_main);
            Misbehaving(pfrom->GetId(), 20, strprintf("message inv size() = %u", vInv.size()));
            return false;
        }

2.  判断是否只允许发送区块。如果是白名单节点，则可以发送区块。

        bool fBlocksOnly = !fRelayTxes;
        if (pfrom->fWhitelisted && gArgs.GetBoolArg("-whitelistrelay", DEFAULT_WHITELISTRELAY))
            fBlocksOnly = false;
        uint32_t nFetchFlags = GetFetchFlags(pfrom);

3.  遍历所有的库存消息，进行如下处理：

    -   调用 `AlreadyHave` 方法，判断是否已经有这个库存。

            bool fAlreadyHave = AlreadyHave(inv);

    -   根据是否为交易消息，对消息标志进行处理。

            if (inv.type == MSG_TX) {
                inv.type |= nFetchFlags;
            }

    -   如果是区块消息，则进行如下处理：

        -   调用 `UpdateBlockAvailability` 方法，更新区块的状态信息。

        -   如果没有这个库存，且 `fImporting` 为假，且不重建索引，且`mapBlocksInFlight`中不包括这个库存，则**调用 `PushMessage` 方法，发送 `getheader` 消息**。

                if (!fAlreadyHave && !fImporting && !fReindex && !mapBlocksInFlight.count(inv.hash)) {
                    connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETHEADERS, chainActive.GetLocator(pindexBestHeader), inv.hash));
                }

    -   否则，把库存消息加入到节点的 `filterInventoryKnown` 集合。如果不是只接收区块数据，且 `fImporting` 为假，且 不是重建索引，且不在 IBD 中，则调用节点的 `AskFor` 方法，进行处理，这个方法的主体是把库存消息加入 `mapAskFor` 集合，除此之外，也进行一些其他处理，在些不详细说明。

            pfrom->AddInventoryKnown(inv);
            if (fBlocksOnly) {
                LogPrint(BCLog::NET, "transaction (%s) inv sent in violation of protocol peer=%d\n", inv.hash.ToString(), pfrom->GetId());
            } else if (!fAlreadyHave && !fImporting && !fReindex && !IsInitialBlockDownload()) {
                pfrom->AskFor(inv);
            }

4.  返回真。


##  11、IBD 时，区块优先节点获取区块消息

在 IBD 期间，节点需要下载大量的区块来使本节点的区块链达到最佳高度。这又分为区块优先和头部优先两种情况，前面讲了头部优先下载，这是当前的主流方式。下面简要讲解下区块优先方式，这是老的客户端工作方式。


### 11.1、`getblocks` 消息

作为服务器，响应客户端节点发送的请求区块消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2006 行。

以 `if (strCommand == NetMsgType::GETBLOCKS) {` 为标志，具体如下：

1.  从输入流中取得区块定位器和结束区块的哈希值。

        CBlockLocator locator;
        uint256 hashStop;
        vRecv >> locator >> hashStop;

2.  如果客户端请求的定位器 `vHave` 数量超过规定的最大值，则设置客户端节点断开，并返回真。

        if (locator.vHave.size() > MAX_LOCATOR_SZ) {
            pfrom->fDisconnect = true;
            return true;
        } 

3.  调用 `ActivateBestChain` 方法，激活最佳的区块链。

4.  调用 `FindForkInGlobalIndex` 方法，查找主链上客户端请求的最后一个区块的哈希值。

        const CBlockIndex* pindex = FindForkInGlobalIndex(chainActive, locator);

    这个方法的实现比较简单：

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

5.  如果需要的最后一个区块存在，那么找到它的下一个区块。

       if (pindex)
            pindex = chainActive.Next(pindex);

6.  进行 for 循环，直到链的最顶端，或达到指定的结束区块。具体处理如下：

    -   如果当前区块就是要停止的区块，则退出循环。

    -   如果处于修剪模式下，并且当前区块索引不在硬盘上或者当前区块索引代码的区块的高度与区块链顶部区块的高度之差大于根据规定计算出来的值，则退出循环。

    -   根据当前的区块索引生成库存对象，并保存到节点的 `vInventoryBlockToSend` 集合中。

    -   如果达到当前最大的区块数量（当前500），则设置节点的继续位置为当前区块的哈希值。

    以上逻辑的代码如下：

        int nLimit = 500;
        for (; pindex; pindex = chainActive.Next(pindex))
        {
            if (pindex->GetBlockHash() == hashStop)
            {
                LogPrint(BCLog::NET, "  getblocks stopping at %d %s\n", pindex->nHeight, pindex->GetBlockHash().ToString());
                break;
            }
            const int nPrunedBlocksLikelyToHave = MIN_BLOCKS_TO_KEEP - 3600 / chainparams.GetConsensus().nPowTargetSpacing;
            if (fPruneMode && (!(pindex->nStatus & BLOCK_HAVE_DATA) || pindex->nHeight <= chainActive.Tip()->nHeight - nPrunedBlocksLikelyToHave))
            {
                break;
            }
            pfrom->PushInventory(CInv(MSG_BLOCK, pindex->GetBlockHash()));
            if (--nLimit <= 0)
            {
                pfrom->hashContinue = pindex->GetBlockHash();
                break;
            }
        }

7.  返回真。

> 注：`inv` 消息由后台线程定时发送。


##  12、紧凑区块中的交易消息处理

### 12.1、`getblocktxn` 消息

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2074 行。

以 `if (strCommand == NetMsgType::GETBLOCKTXN) {` 为标志，具体如下：

1.  从输入流中取得请求对象。

        BlockTransactionsRequest req;
        vRecv >> req;

2.  如果请求的交易所在的区块就是最近的区块，则设置变量 `recent_block`。如果变量不空，则调用 `SendBlockTransactions` 方法，发送区块的交易，然后返回真。

        std::shared_ptr<const CBlock> recent_block;
        {
            LOCK(cs_most_recent_block);
            if (most_recent_block_hash == req.blockhash)
                recent_block = most_recent_block;
        }
        if (recent_block) {
            SendBlockTransactions(*recent_block, req, pfrom, connman);
            return true;
        }


    `SendBlockTransactions` 方法中构造一个响应对象，把区块中的所有交易加入到响应对象的 `txn` 集合中，然后调用 `PushMessage` 方法，发送 `blocktxn` 消息。

3.  调用 `LookupBlockIndex` 方法，查找客户端请求的区块索引。如果不存在请求的索引，或者索引对应的区块数据不在硬盘上，则直接返回。

        const CBlockIndex* pindex = LookupBlockIndex(req.blockhash);
        if (!pindex || !(pindex->nStatus & BLOCK_HAVE_DATA)) {
            return true;
        }

4.  如果请求区块的高度与区块链顶部的高度之差大于规定的最大深度 10 （`MAX_BLOCKTXN_DEPTH`），则生成一个库存对象，并加入节点的 `vRecvGetData` 集合中，然后返回真。`vRecvGetData` 集合中的数据由定时器定时处理。

        if (pindex->nHeight < chainActive.Height() - MAX_BLOCKTXN_DEPTH) {
            CInv inv;
            inv.type = State(pfrom->GetId())->fWantsCmpctWitness ? MSG_WITNESS_BLOCK : MSG_BLOCK;
            inv.hash = req.blockhash;
            pfrom->vRecvGetData.push_back(inv);
            return true;
        }

5.  以上情况都不是，则从硬盘中加载区块，并进行发送。

        CBlock block;
        bool ret = ReadBlockFromDisk(block, pindex, chainparams.GetConsensus());
        SendBlockTransactions(block, req, pfrom, connman);

6.  返回真。


### 12.2、`blocktxn` 消息

作为客户端，处理服务器节点发送的区块中的交易。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2598 行。

以 `if (strCommand == NetMsgType::BLOCKTXN && !fImporting && !fReindex)` 为标志，具体如下：


1.  从输入流中取得服务器发送的交易。

        BlockTransactions resp;
        vRecv >> resp;

2.  如果接收到的交易所在的区块不存在于 `mapBlocksInFlight` 集合中，或者不是当前节点，则直接退出，并返回真。

        std::shared_ptr<CBlock> pblock = std::make_shared<CBlock>();
        bool fBlockRead = false;

        std::map<uint256, std::pair<NodeId, std::list<QueuedBlock>::iterator> >::iterator it = mapBlocksInFlight.find(resp.blockhash);
        if (it == mapBlocksInFlight.end() || !it->second.second->partialBlock ||
                it->second.first != pfrom->GetId()) {
            return true;
        }

3.  从部分下载的区块中填充区块对象。如果服务器发送的区块是无效的，那么调用 `Misbehaving` 方法，惩罚远程节点；否则，如果不能处理区块数据，则生成一个库存对象，然后调用 `PushMessage` 消息，重新发送 `getdata` 消息；否则，即发送的区块是OK的，则调用 `MarkBlockAsReceived` 方法，设置节点状态对象的相关属性，同时把接收到的区块相关内容保存在 `mapBlockSource` 集合中。

        PartiallyDownloadedBlock& partialBlock = *it->second.second->partialBlock;
        ReadStatus status = partialBlock.FillBlock(*pblock, resp.txn);
        if (status == READ_STATUS_INVALID) {
            MarkBlockAsReceived(resp.blockhash); // Reset in-flight state in case of whitelist
            Misbehaving(pfrom->GetId(), 100, strprintf("Peer %d sent us invalid compact block/non-matching block transactions\n", pfrom->GetId()));
            return true;
        } else if (status == READ_STATUS_FAILED) {
            std::vector<CInv> invs;
            invs.push_back(CInv(MSG_BLOCK | GetFetchFlags(pfrom), resp.blockhash));
            connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::GETDATA, invs));
        } else {
            MarkBlockAsReceived(resp.blockhash); // it is now an empty pointer
            fBlockRead = true;
            mapBlockSource.emplace(resp.blockhash, std::make_pair(pfrom->GetId(), false));
        }

4.  如果区块数据可以读取，那么调用 `ProcessNewBlock` 方法，处理收到的区块。如果这个区块是新的，则修改节点的最后一次接收到区块的时间，否则从 `mapBlockSource` 集合中，移除区块对应的数据。

        if (fBlockRead) {
            bool fNewBlock = false;
            ProcessNewBlock(chainparams, pblock, /*fForceProcessing=*/true, &fNewBlock);
            if (fNewBlock) {
                pfrom->nLastBlockTime = GetTime();
            } else {
                LOCK(cs_main);
                mapBlockSource.erase(pblock->GetHash());
            }
        }

5.  返回真。


##  13、`mempool` 消息

作为服务器，处理客户端节点发送的设置内存池消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2598 行。

以 `if (strCommand == NetMsgType::MEMPOOL) {` 为标志，具体如下：


1.  如果节点没有能力也没有意愿启用 bloom-filtered 连接，并且节点也不在白名单中，则断开节点，并返回真。

        if (!(pfrom->GetLocalServices() & NODE_BLOOM) && !pfrom->fWhitelisted)
        {
            pfrom->fDisconnect = true;
            return true;
        }

2.  如果内存池请求达到限额，并且节点也不在白名单中，则断开节点，并返回真。

        if (connman->OutboundTargetReached(false) && !pfrom->fWhitelisted)
        {
            pfrom->fDisconnect = true;
            return true;
        }

3.  设置节点可以发送内存池消息，并返回真。

        pfrom->fSendMempool = true;
        return true;

