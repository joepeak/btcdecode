#   P2P 网络建立之消息处理上篇

P2P 网络的建立是在系统启动的第 12 步，最后时刻调用 `CConnman::Start` 方法开始的。

本部分内容在 `net.cpp`、`net_processing.cpp` 等文件中。

终于，来到了我们非常非常关心比特币消息处理，通过比特币消息处理，我们会理解比特币的协义，理解比特币是如何同步区块，如何发送交易，从而建立起理解比特币的至关重要一步。

本部分内容是如此的重要，也是相当的长，所以我们分上下两部分来介绍具体的消息处理。

上篇主要以消息处理线程的分析为主，下篇以具体的比特币消息即比特币协义分析为主。

下面我们来看消息处理线程相关的代码。


##  ThreadMessageHandler

处理所有消息的线程。主体是一个 `while` 循环，退出条件为 `flagInterruptMsgProc` 为假，循环体如下：

1.  生成一个对等节点容器对象 `vNodesCopy`，类型为 `CNode*`。然后，设置其值为 `vNodes` 。后者，代表当前节点保存的所有对等节点信息。

2.  遍历所有的 `vNodesCopy` 节点，调用当前节点的 `AddRef` 方法，把对等节点的 `nRefCount` 属性加1。

3.  遍历所有的 `vNodesCopy` 节点，进行如下处理：

    -   如果当前节点已断开连接，则处理下一个。

    -   **调用 `PeerLogicValidation::ProcessMessages` 方法，处理当前远程对等节点发送给本对等节点的消息**。

    -   **调用 `PeerLogicValidation::SendMessages` 方法，处理本对等节点发送给当前远程对等节点的消息**。

3.  遍历所有的 `vNodesCopy` 节点，调用当前节点的 `Release` 方法，把对等节点的 `nRefCount` 属性减1。


### 1、ProcessMessages

本方法主要处理对等节点接收到的相关消息，具体代码在 `net_processing.cpp` 文件中。

1.  如果对等节点的已接收请求数据集合不为空，也就是保存别的对等节点请求数据的集合不为空，则调用 `ProcessGetData` 方法，处理别的对等获取数据的请求。

        if (!pfrom->vRecvGetData.empty())
            ProcessGetData(pfrom, chainparams, connman, interruptMsgProc);

    `vRecvGetData` 属性是一个 inv 消息（`CInv`）的双端队列。

    下面，我们来看 `ProcessGetData` 方法是怎样处理收到的消息。

    -   从对等节点的 `vRecvGetData` 集合中，取得其迭代器。

        std::deque<CInv>::iterator it = pfrom->vRecvGetData.begin();
        std::vector<CInv> vNotFound;

    -   生成一个 `CNetMsgMaker` 类型的对象 `msgMaker`。其 `nVersionIn` 属性是对等节点已经发送的版本消息。

    -   遍历请求数据集合，如果请求数据的类型是交易或者见证隔离交易，进行如下的处理：

        如果已经终止处理消息信号为真，则直接返回。如果当前节点处于暂停状态（缓冲区太慢而不能响应），则推出循环。

        取得当前的 `inv` 消息。从 `mapRelay` 集合中取得当前 `inv` 消息。这个 `mapRelay` 是一个 Map 集合，Key 是交易的哈希，Value 是指向交易的智能指针。

        如果查找的交易存在于集合中，则调用位于 `net.cpp` 文件中的 `CConnman::PushMessage` 方法，发送交易；否则，如果节点最后一次 `MEMPOOL` 请求存在，则从内存池中取得对应的交易信息，然后调用 `CConnman::PushMessage` 方法，发送交易。

            while (it != pfrom->vRecvGetData.end() && (it->type == MSG_TX || it->type == MSG_WITNESS_TX)) {
                if (interruptMsgProc)
                    return;
                // Don't bother if send buffer is too full to respond anyway
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

    -   如果没有达到请求数据集合尾部，且节点对象不是暂停状态，并且当前请求的 `inv` 消息类型是区块、过滤型区块、紧凑型区块、隔离见证型区块之一，则调用 `ProcessGetBlockData` 方法处理区块数据。 遍历收到的所有数据，如果数据类型是交易或者见证隔离交易，进行如下的处理：

        `ProcessGetBlockData` 方法，因为处理过程比较长，所以其处理过程放在下面进行详细说明。

    -   清空已收到数据集合从起始部分到当前位置之间的 `inv` 数据。

2.  如果对等节点已断开，则直接返回。

3.  如果对等节点要处理的数据不为空，则直接返回。

    这样维护了响应的顺序。

4.  如果对等节点已暂停，则直接返回。

    缓冲区太满而不能进行继续处理。

5.  如果节点对象待处理的消息列表 `vProcessMsg` 为空，则返回假。否则，取出第一个消息对象，放入消息列表 `msgs` 中，并待处理的消息列表中删除。将节点对象处理队列大小减去已删除的消息对象收到的数据长度与消息头部大小之和。根据节点对象处理队列大小与允许的接收上限比较，如果大于接收上限，则设置节点暂停接收消息。

        if (pfrom->vProcessMsg.empty())
            return false;
        msgs.splice(msgs.begin(), pfrom->vProcessMsg, pfrom->vProcessMsg.begin());
        pfrom->nProcessQueueSize -= msgs.front().vRecv.size() + CMessageHeader::HEADER_SIZE;
        pfrom->fPauseRecv = pfrom->nProcessQueueSize > connman->GetReceiveFloodSize();
        fMoreWork = !pfrom->vProcessMsg.empty();

6.  生成一个消息对象，设置其版本为对等节点已接收到的版本。

        CNetMessage& msg(msgs.front());
        msg.SetVersion(pfrom->GetRecvVersion());

7.  验证消息的 `MESSAGESTART` 是否有效。如果无效，则设置对等节点为断开，然后返回。

        if (memcmp(msg.hdr.pchMessageStart, chainparams.MessageStart(), CMessageHeader::MESSAGE_START_SIZE) != 0) {
            LogPrint(BCLog::NET, "PROCESSMESSAGE: INVALID MESSAGESTART %s peer=%d\n", SanitizeString(msg.hdr.GetCommand()), pfrom->GetId());
            pfrom->fDisconnect = true;
            return false;
        }

8.  从消息对象中取得消息头部，并进行验证。如果无效，则返回。

        CMessageHeader& hdr = msg.hdr;
        if (!hdr.IsValid(chainparams.MessageStart()))
        {
            LogPrint(BCLog::NET, "PROCESSMESSAGE: ERRORS IN HEADER %s peer=%d\n", SanitizeString(hdr.GetCommand()), pfrom->GetId());
            return fMoreWork;
        }

9.  从消息对象中取得具体的命令和消息大小。

        std::string strCommand = hdr.GetCommand();
        unsigned int nMessageSize = hdr.nMessageSize;

10.  调用消息对象自身的哈希，并与消息头部的校验和进行验证。如果验证失败，则返回。

        const uint256& hash = msg.GetMessageHash();
        if (memcmp(hash.begin(), hdr.pchChecksum, CMessageHeader::CHECKSUM_SIZE) != 0)
        {
            LogPrint(BCLog::NET, "%s(%s, %u bytes): CHECKSUM ERROR expected %s was %s\n", __func__,
               SanitizeString(strCommand), nMessageSize,
               HexStr(hash.begin(), hash.begin()+CMessageHeader::CHECKSUM_SIZE),
               HexStr(hdr.pchChecksum, hdr.pchChecksum+CMessageHeader::CHECKSUM_SIZE));
            return fMoreWork;
        }

11. **调用 `ProcessMessage` 方法，进行消息处理**。

    对于比特币网络来说，最最重要的方法来了。这个方法处理比特币的所有具体，比如：版本消息、获取区块消息等。

    因为这个方法是如此的重要，所以我们把留在下一篇文章中进行说明。

11. 调用 `SendRejectsAndCheckIfBanned` 方法，进行可能的 `reject` 处理。具体如下：

    -   取得节点的状态对象。

    -   如果启用 BIP61，则遍历状态对象中保存的 `reject` ，调用 `PushMessage` 方法，发送 `reject` 消息。

    -   清空状态对象中保存的 `reject`.

    -   如果状态对象的禁止属性 `fShouldBan` 为真，则：

        -   设置禁止属性为假。

        -   如果节点在白名单中，或者是手动连接的，则进行警告。否则，进行下面的处理：

        -   设置对等节点的断开连接属性为真；如果节点的地址是本地地址，则进行警告，否则，调用 `Ban` 方法，禁止对等节点连接。


####    ProcessGetBlockData

1.  生成一些内部变量，并设置为已缓存的变量值。

2.  调用 `LookupBlockIndex` 方法，查询消息对应的区块索引。如果区块索引存在，并且索引对应的区块在链上的交易也存在，但区块还没有验证过，那么设置变量 `need_activate_chain` 为真。

    const CBlockIndex* pindex = LookupBlockIndex(inv.hash);
    if (pindex) {
        if (pindex->nChainTx && !pindex->IsValid(BLOCK_VALID_SCRIPTS) && pindex->IsValid(BLOCK_VALID_TREE)) {
            need_activate_chain = true;
        }
    }

3.  如果需要激活区块链，那么调用 `ActivateBestChain` 方法来激活。

4.  如果区块索引存在，调用 `BlockRequestAllowed` 方法检查是否允许发送数据。

5.  如果允许发送，且达到历史区块服务限额的情况下，断开节点连接，并设置发送标志为假。

        if (send && connman->OutboundTargetReached(true) && ( ((pindexBestHeader != nullptr) && (pindexBestHeader->GetBlockTime() - pindex->GetBlockTime() > HISTORICAL_BLOCK_AGE)) || inv.type == MSG_FILTERED_BLOCK) && !pfrom->fWhitelisted)
        {
            LogPrint(BCLog::NET, "historical block serving limit reached, disconnect peer=%d\n", pfrom->GetId());
            pfrom->fDisconnect = true;
            send = false;
        }

6.  如果允许发送，且节点不在白名单中，且节点支持的服务是 `NODE_NETWORK_LIMITED`（只支持 288个区块，即2天内生成的区块），且不支持 `NODE_NETWORK` 服务，且区块链栈顶的高度与当前区块索引的高度之差大于网络限制允许的最小区块数（`NODE_NETWORK_LIMITED_MIN_BLOCKS`，288个区块）加上额外的两个区块（为了防止竞争，指的是分叉？所以增加两个区块缓冲），则断开节点连接，并设置发送标志为假。

        if (send && !pfrom->fWhitelisted && (
                (((pfrom->GetLocalServices() & NODE_NETWORK_LIMITED) == NODE_NETWORK_LIMITED) &&
                 ((pfrom->GetLocalServices() & NODE_NETWORK) != NODE_NETWORK) &&
                 (chainActive.Tip()->nHeight - pindex->nHeight > (int)NODE_NETWORK_LIMITED_MIN_BLOCKS + 2) )
           )) {
            pfrom->fDisconnect = true;
            send = false;
        }

7.  如果允许发送，并且这个区块索引的状态等于 `BLOCK_HAVE_DATA` （全部数据都 `blk*.dat` 文件中可用），则进行下面的处理：

    -   生成一个指向区块对象的智能指针对象 `pblock`。

    -   如果最近区块对象存在，且其哈希与区块索引对象的哈希一样，那么设置 `pblock` 为最近区块对象；

    -   否则，如果消息对象类型是隔离见证区块，那么：调用 `ReadRawBlockFromDisk` 方法，从磁盘中读取原始的区块数据。如果可以读到，则调用 `PushMessage` 方法，发送区块数据。这种情况下，直接发送了区块数据，所以不设置 `pblock` 变量。

    -   否则，调用 `ReadBlockFromDisk` 方法，从磁盘中读取原始的区块数据。如果可以读到，则设置 `pblock` 为读取到的数据。

    -   如果 `pblock` 为真，则发送消息。

        如果消息对象类型是区块，则调用 `PushMessage` 方法，发送标志为 `SERIALIZE_TRANSACTION_NO_WITNESS` 的区块消息；

        如果消息类型是隔离见证区块，则调用 `PushMessage` 方法，发送区块消息；

        如果消息类型是过滤区块，则：如果对节对象的 Bloom 过滤器存在，那么生成默克尔区块，并设置发送过滤区块标志为真；如果发送过滤区块标志为真，则调用 `PushMessage` 方法，发送默克尔区块，然后调用 `PushMessage` 方法，发送默克尔区块的每个交易数据，标志为 `SERIALIZE_TRANSACTION_NO_WITNESS`；

        如果消息类型是紧凑区块，同样调用 `PushMessage` 方法，发送区块消息。

    -   如果消息的哈希等于节点的继续发送属性（`hashContinue` 属性，代表继续发送的哈希），则：

        生成一个 `CInv` 向量；然后，构造一个 `CInv` 对象，类型为区块，哈希为区块链栈顶元素的哈希；然后，调用 `PushMessage` 方法，发送 `inv` 消息；最后，设置节点的继续发送属性为空。

    代码如下：

        if (send && (pindex->nStatus & BLOCK_HAVE_DATA)){
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
            if (pblock) {
                if (inv.type == MSG_BLOCK)
                    connman->PushMessage(pfrom, msgMaker.Make(SERIALIZE_TRANSACTION_NO_WITNESS, NetMsgType::BLOCK, *pblock));
                else if (inv.type == MSG_WITNESS_BLOCK)
                    connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::BLOCK, *pblock));
                else if (inv.type == MSG_FILTERED_BLOCK)
                {
                    bool sendMerkleBlock = false;
                    CMerkleBlock merkleBlock;
                    {
                        LOCK(pfrom->cs_filter);
                        if (pfrom->pfilter) {
                            sendMerkleBlock = true;
                            merkleBlock = CMerkleBlock(*pblock, *pfrom->pfilter);
                        }
                    }
                    if (sendMerkleBlock) {
                        connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::MERKLEBLOCK, merkleBlock));
                        typedef std::pair<unsigned int, uint256> PairType;
                        for (PairType& pair : merkleBlock.vMatchedTxn)
                            connman->PushMessage(pfrom, msgMaker.Make(SERIALIZE_TRANSACTION_NO_WITNESS, NetMsgType::TX, *pblock->vtx[pair.first]));
                    }
                }
                else if (inv.type == MSG_CMPCT_BLOCK)
                {
                    bool fPeerWantsWitness = State(pfrom->GetId())->fWantsCmpctWitness;
                    int nSendFlags = fPeerWantsWitness ? 0 : SERIALIZE_TRANSACTION_NO_WITNESS;
                    if (CanDirectFetch(consensusParams) && pindex->nHeight >= chainActive.Height() - MAX_CMPCTBLOCK_DEPTH) {
                        if ((fPeerWantsWitness || !fWitnessesPresentInARecentCompactBlock) && a_recent_compact_block && a_recent_compact_block->header.GetHash() == pindex->GetBlockHash()) {
                            connman->PushMessage(pfrom, msgMaker.Make(nSendFlags, NetMsgType::CMPCTBLOCK, *a_recent_compact_block));
                        } else {
                            CBlockHeaderAndShortTxIDs cmpctblock(*pblock, fPeerWantsWitness);
                            connman->PushMessage(pfrom, msgMaker.Make(nSendFlags, NetMsgType::CMPCTBLOCK, cmpctblock));
                        }
                    } else {
                        connman->PushMessage(pfrom, msgMaker.Make(nSendFlags, NetMsgType::BLOCK, *pblock));
                    }
                }
            }
            if (inv.hash == pfrom->hashContinue)
            {
                std::vector<CInv> vInv;
                vInv.push_back(CInv(MSG_BLOCK, chainActive.Tip()->GetBlockHash()));
                connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::INV, vInv));
                pfrom->hashContinue.SetNull();
            }
        }


### 2、SendMessages

本方法主要处理对等节点发送的相关逻辑，具体代码在 `net_processing.cpp` 文件中。

1.  如果对等节点间还没有完成握手，或者已经断开连接，则返回。

        if (!pto->fSuccessfullyConnected || pto->fDisconnect)
            return true;

2.  如果对等节点的 Ping 已经被请求，则设置 `pingSend` 变量为真。

3.  如果对等节点没有期望 Pong 回复（即对等节点的 `nPingNonceSent` 等于0），且用户开始 ping 的时间（`nPingUsecStart` ）加上规定的节点 ping 间隔小于当前时间，则设置 `pingSend` 变量为真。

4.  如果 `pingSend` 为真，处理如下：

    -   设置对等节点的 Ping 已经被请求假。

    -   设置开始 ping 的时间为当前时间。

    -   如果对等节点的版本大于 BIP 0031 规定的版本（60000），则设置对等节点的 `nPingNonceSent` 为随机变量 `nonce`，调用 `PushMessage` 方法，发送 `ping` 消息，消息中包括 `nonce`；否则，即对等节点不支持带随机数的 Ping 命令，则设置设置对等节点的 `nPingNonceSent` 为0，调用 `PushMessage` 方法，发送 `ping` 消息，消息中不包括 `nonce`。

    if (pingSend) {
        uint64_t nonce = 0;
        while (nonce == 0) {
            GetRandBytes((unsigned char*)&nonce, sizeof(nonce));
        }
        pto->fPingQueued = false;
        pto->nPingUsecStart = GetTimeMicros();
        if (pto->nVersion > BIP0031_VERSION) {
            pto->nPingNonceSent = nonce;
            connman->PushMessage(pto, msgMaker.Make(NetMsgType::PING, nonce));
        } else {
            pto->nPingNonceSent = 0;
            connman->PushMessage(pto, msgMaker.Make(NetMsgType::PING));
        }
    }

5.  调用 `SendRejectsAndCheckIfBanned` 方法，进行可能的 `reject` 处理。如果该函数返回为真，则返回。

6.  获取节点的状态对象。

7.  如果当前没有在 IBD 下载中（`IsInitialBlockDownload` 函数为假），且节点下次发送本地地址的时间（`nNextLocalAddrSend`）小于当前时间，那么进行如下处理：

    -   调用 `AdvertiseLocal` 方法，发送我们自己本身的地址给对等节点。

        方法的主要逻辑是调用节点对象的 `PushAddress` 方法，把要发送的地址保存在 `vAddrToSend` 集合中。

    -   调用 `PoissonNextSend` 方法，计算下次发送地址的时间，并设置节点的下次发送本地地址的时间为该值。

7.  如果节点的下次发送地址时间小于当前时间，则：

    -   调用 `PoissonNextSend` 方法，计算下次发送地址的时间，并保存为节点的下次发送地址时间 `nNextAddrSend` 属性。

    -   生成一个地址向量集合 `vAddr`。

    -   遍历节点待发送的地址向量 `vAddrToSend`，如果当前地址不在节点的概率“跟踪最近插入的”集合`addrKnown` 中，则：

        -   保存当前地址到节点的概率“跟踪最近插入的”集合中。

        -   保存地址到 `vAddr` 向量中。

        -   如果当前的地址向量集合数量大于等于 1000，则调用 `PushMessage` 方法，发送地址向量；然后清空地址向量集合。

    -   清空节点的发送地址集合 `vAddrToSend`。

    -   如果地址向量集合不空，即地址向量集合的数量不超过 1000个，则调用 `PushMessage` 方法，发送地址消息。

    -   如果节点的待发送的地址向量集合的预分配的内存空间（`capacity()`）大于40，则调用其 `shrink_to_fit` 方法来缩减空间，即只允许发送一次大的地址包。

    if (pto->nNextAddrSend < nNow) {
        pto->nNextAddrSend = PoissonNextSend(nNow, AVG_ADDRESS_BROADCAST_INTERVAL);
        std::vector<CAddress> vAddr;
        vAddr.reserve(pto->vAddrToSend.size());
        for (const CAddress& addr : pto->vAddrToSend)
        {
            if (!pto->addrKnown.contains(addr.GetKey()))
            {
                pto->addrKnown.insert(addr.GetKey());
                vAddr.push_back(addr);
                // receiver rejects addr messages larger than 1000
                if (vAddr.size() >= 1000)
                {
                    connman->PushMessage(pto, msgMaker.Make(NetMsgType::ADDR, vAddr));
                    vAddr.clear();
                }
            }
        }
        pto->vAddrToSend.clear();
        if (!vAddr.empty())
            connman->PushMessage(pto, msgMaker.Make(NetMsgType::ADDR, vAddr));
        if (pto->vAddrToSend.capacity() > 40)
            pto->vAddrToSend.shrink_to_fit();
    }

8.  接下来，开始区块同步。

    -   如果指向最佳区块链头部的指针（`pindexBestHeader`）为空指针，则设置其为当前活跃区块链的栈顶元素指针。

            if (pindexBestHeader == nullptr)
                pindexBestHeader = chainActive.Tip();

    -   如果还没有从这个节点同步区块头部，并且节点的 `fClient`、`fImporting`、`fReindex` 等属性为假，进一步，如果已经同步的节点数量为 0 且需要获取区块数据，或者最佳区块头部的区块时间距离现在已超过 24 小时，那么调用 `PushMessage` 方法，发出请求 `GETHEADERS` 命令，开始同步区块头部。

        bool fFetch = state.fPreferredDownload || (nPreferredDownload == 0 && !pto->fClient && !pto->fOneShot); // Download if this is a nice peer, or we have no nice peers and this one might do.
        if (!state.fSyncStarted && !pto->fClient && !fImporting && !fReindex) {
            if ((nSyncStarted == 0 && fFetch) || pindexBestHeader->GetBlockTime() > GetAdjustedTime() - 24 * 60 * 60) {
                state.fSyncStarted = true;
                state.nHeadersSyncTimeout = GetTimeMicros() + HEADERS_DOWNLOAD_TIMEOUT_BASE + HEADERS_DOWNLOAD_TIMEOUT_PER_HEADER * (GetAdjustedTime() - pindexBestHeader->GetBlockTime())/(consensusParams.nPowTargetSpacing);
                nSyncStarted++;
                const CBlockIndex *pindexStart = pindexBestHeader;
                if (pindexStart->pprev)
                    pindexStart = pindexStart->pprev;
                LogPrint(BCLog::NET, "initial getheaders (%d) to peer=%d (startheight:%d)\n", pindexStart->nHeight, pto->GetId(), pto->nStartingHeight);
                connman->PushMessage(pto, msgMaker.Make(NetMsgType::GETHEADERS, chainActive.GetLocator(pindexStart), uint256()));
            }
        }

9.  如果当前不是重建索引、重新导入和 IBD 下载期间，则重新发送尚未进入区块的钱包交易。

        if (!fReindex && !fImporting && !IsInitialBlockDownload())
        {
            GetMainSignals().Broadcast(nTimeBestReceived, connman);
        }

10. 如果不需要转化为 `Inv` 消息，那么进行如下处理：

    遍历区块头部进行区块公告的集合（`vBlockHashesToAnnounce`），按如下处理：

    -   调用 `LookupBlockIndex` 方法，查找当前对应的区块索引。

    -   如果当前活跃区块链上没有这个索引，那么设置需要转化为 `Inv` 消息为真，然后退出当前循环。

            const CBlockIndex* pindex = LookupBlockIndex(hash);
            if (chainActive[pindex->nHeight] != pindex) {
                fRevertToInv = true;
                break;
            }

    -   如果最佳索引不是空指针，并且当前区块索引不等于最佳指针，那么设置需要转化为 `Inv` 消息为真，然后退出当前循环。

            if (pBestIndex != nullptr && pindex->pprev != pBestIndex) {
                fRevertToInv = true;
                break;
            }

    -   接下来，设置当前区块索引为最佳索引，处理哪些区块索引可以放入头部集合。

            pBestIndex = pindex;
            if (fFoundStartingHeader) {
                // add this to the headers message
                vHeaders.push_back(pindex->GetBlockHeader());
            } else if (PeerHasHeader(&state, pindex)) {
                continue; // keep looking for the first new block
            } else if (pindex->pprev == nullptr || PeerHasHeader(&state, pindex->pprev)) {
                // Peer doesn't have this header but they do have the prior one.
                // Start sending headers.
                fFoundStartingHeader = true;
                vHeaders.push_back(pindex->GetBlockHeader());
            } else {
                // Peer doesn't have this header or the prior one -- nothing will
                // connect, so bail out.
                fRevertToInv = true;
                break;
            }

11. 如果不需要转化成 `Inv` 消息，并且区块头部集合（`vHeaders`）不空，进行下面的处理：

    -   如果区块头部长度为1，并且需要下载头部和ID，那么：

        生成发送标志，如果对等节点想要紧凑的隔离见证类型，则设置发送标志为 0，否则设置为 `SERIALIZE_TRANSACTION_NO_WITNESS`。

        如果最近的区块哈希与最佳区块索引的哈希相等，则调用 `PushMessage` 方法，发送消息，类型为 `CMPCTBLOCK`，然后设置区块是从缓存区中加载的标志为真。

        如果区块不是从缓存区中加载的，那么就需要调用 `ReadBlockFromDisk` 方法，从硬盘中加载区块，然后再调用 `PushMessage` 方法，发送消息，类型为 `CMPCTBLOCK`。

    -   否则，如果不是优先下载头部（即区块状态对象 `fPreferHeaders`）为假，那么就调用 `PushMessage` 方法，发送消息，类型为 `HEADERS`，然后设置区块状态对象的 `pindexBestHeaderSent` 属性为当前的最佳索引区块。

    具体代码如下：

        if (!fRevertToInv && !vHeaders.empty()) {
            if (vHeaders.size() == 1 && state.fPreferHeaderAndIDs) {
                int nSendFlags = state.fWantsCmpctWitness ? 0 : SERIALIZE_TRANSACTION_NO_WITNESS;
                bool fGotBlockFromCache = false;
                {
                    LOCK(cs_most_recent_block);
                    if (most_recent_block_hash == pBestIndex->GetBlockHash()) {
                        if (state.fWantsCmpctWitness || !fWitnessesPresentInMostRecentCompactBlock)
                            connman->PushMessage(pto, msgMaker.Make(nSendFlags, NetMsgType::CMPCTBLOCK, *most_recent_compact_block));
                        else {
                            CBlockHeaderAndShortTxIDs cmpctblock(*most_recent_block, state.fWantsCmpctWitness);
                            connman->PushMessage(pto, msgMaker.Make(nSendFlags, NetMsgType::CMPCTBLOCK, cmpctblock));
                        }
                        fGotBlockFromCache = true;
                    }
                }
                if (!fGotBlockFromCache) {
                    CBlock block;
                    bool ret = ReadBlockFromDisk(block, pBestIndex, consensusParams);
                    assert(ret);
                    CBlockHeaderAndShortTxIDs cmpctblock(block, state.fWantsCmpctWitness);
                    connman->PushMessage(pto, msgMaker.Make(nSendFlags, NetMsgType::CMPCTBLOCK, cmpctblock));
                }
                state.pindexBestHeaderSent = pBestIndex;
            } else if (state.fPreferHeaders) {
                connman->PushMessage(pto, msgMaker.Make(NetMsgType::HEADERS, vHeaders));
                state.pindexBestHeaderSent = pBestIndex;
            } else
                fRevertToInv = true;
        }


12. 如果需要转化成 `Inv` 消息，进一步，如果使用区块头部进行区块公告的集合（`vBlockHashesToAnnounce`），则进行如下处理：

    -   返回集合最后一个元素，调用 `LookupBlockIndex` 方法，查找这个元素对应的区块索引。

    -   如果活跃区块链在区块索引指定的高度上对应的索引不是我们找到的索引，即要公告的区块不在主链上，则打印一个警告。

    -   如果这个区块不在节点的区块链上，那么就把这个区块放在区块库存清单中。

        生成一个 `Inv` 消息，类型是区块，然后调用节点对象的 `PushInventory` 方法，放入节点对象的库存清单集合中。

13. 清空节点对象的区块头部进行区块公告的集合。

14. 生成 `vInv` 集合，并设置其长度。

        std::vector<CInv> vInv;
        vInv.reserve(std::max<size_t>(pto->vInventoryBlockToSend.size(), INVENTORY_BROADCAST_MAX));

15. 遍历已经公告的区块 ID 列表，如果没有达到 `vInv` 集合的最大长度，则加入集合尾部，如果已经达到则调用 `PushMessage` 方法，发送 `INV` 消息。然后，清空已经公告的区块 ID 列表。

        for (const uint256& hash : pto->vInventoryBlockToSend) {
            vInv.push_back(CInv(MSG_BLOCK, hash));
            if (vInv.size() == MAX_INV_SZ) {
                connman->PushMessage(pto, msgMaker.Make(NetMsgType::INV, vInv));
                vInv.clear();
            }
        }
        pto->vInventoryBlockToSend.clear();

16. 如果节点对象下次发送 `Inv` 消息的时间已经小于当前时间，那么设置 `fSendTrickle` 变量为真，根据是否为入门节点，设置不同的节点对象下次发送 `Inv` 消息。

        bool fSendTrickle = pto->fWhitelisted;
        if (pto->nNextInvSend < nNow) {
            fSendTrickle = true;
            if (pto->fInbound) {
                pto->nNextInvSend = connman->PoissonNextSendInbound(nNow, INVENTORY_BROADCAST_INTERVAL);
            } else {
                pto->nNextInvSend = PoissonNextSend(nNow, INVENTORY_BROADCAST_INTERVAL >> 1);
            }
        }

17. 如果发送时间已到，但是节点请求我们不要发送中继交易，那么清空节点对象的发送 `Inv` 消息的集合集合 `setInventoryTxToSend`。

        if (fSendTrickle) {
            LOCK(pto->cs_filter);
            if (!pto->fRelayTxes) pto->setInventoryTxToSend.clear();
        }

18. 如果发送时间已到，并且节点请求过 BIP35 规定的 `mempool` ，那么：

    -   调用内存池对象的 `infoAll` 方法，返回内存池交易信息集合。

    -   设置区块对象的 `mempool` 为假。

    -   获取节点设置的最小交易费用过滤器。默认为 0。

            CAmount filterrate = 0;
            {
                LOCK(pto->cs_feeFilter);
                filterrate = pto->minFeeFilter;
            }

    -   遍历内存池交易信息集合并进行处理。

        用当前交易信息生成一个 inv 对象，然后从区块对象的 `setInventoryTxToSend` 集合中删除对应的交易信息。如果设置了最小交易费用，并且当前交易的费用小于设置的最小费用，那么处理下一个。

            const uint256& hash = txinfo.tx->GetHash();
            CInv inv(MSG_TX, hash);
            pto->setInventoryTxToSend.erase(hash);
            if (filterrate) {
                if (txinfo.feeRate.GetFeePerK() < filterrate)
                    continue;
            }

        如果区块对象设置了布隆过滤器，并且当前交易不符合要求，那么处理下一个。

            if (pto->pfilter) {
                if (!pto->pfilter->IsRelevantAndUpdate(*txinfo.tx)) continue;
            }

        把当前交易的哈希加入区块对象的 `filterInventoryKnown` 集合；把 inv 对象加入 `vInv` 集合。如果集合已经达到规定的最大数量，那么调用 `PushMessage` 方法，发送 `INV` 消息给远程对等节点，然后清空集合。

            pto->filterInventoryKnown.insert(hash);
            vInv.push_back(inv);
            if (vInv.size() == MAX_INV_SZ) {
                connman->PushMessage(pto, msgMaker.Make(NetMsgType::INV, vInv));
                vInv.clear();
            }
    
    -   设置区块对象的 `timeLastMempoolReq` 属性。

19. 如果需要发送，即发送时间已到，那么：

    -   生成交易的向量集合，并设置其大小。然后从区块对象的 `setInventoryTxToSend` 集合中取得其迭代器放进新生成的向量集合。

            std::vector<std::set<uint256>::iterator> vInvTx;
            vInvTx.reserve(pto->setInventoryTxToSend.size());
            for (std::set<uint256>::iterator it = pto->setInventoryTxToSend.begin(); it != pto->setInventoryTxToSend.end(); it++) {
                vInvTx.push_back(it);
            }

    -   设置区块对象的最小交易费用，默认为0。

            CAmount filterrate = 0;
            {
                LOCK(pto->cs_feeFilter);
                filterrate = pto->minFeeFilter;
            }

    -   如果 `vInvTx` 集合不空，并且需要中继的交易数量小于规定的最大 INV 广播数量，那就进行 `while` 循环。下面是循环体：

            while (!vInvTx.empty() && nRelayedTransactions < INVENTORY_BROADCAST_MAX) {
                std::pop_heap(vInvTx.begin(), vInvTx.end(), compareInvMempoolOrder);
                std::set<uint256>::iterator it = vInvTx.back();
                vInvTx.pop_back();
                uint256 hash = *it;
                pto->setInventoryTxToSend.erase(it);
                if (pto->filterInventoryKnown.contains(hash)) {
                    continue;
                }
                auto txinfo = mempool.info(hash);
                if (!txinfo.tx) {
                    continue;
                }
                if (filterrate && txinfo.feeRate.GetFeePerK() < filterrate) {
                    continue;
                }
                if (pto->pfilter && !pto->pfilter->IsRelevantAndUpdate(*txinfo.tx)) continue;
                vInv.push_back(CInv(MSG_TX, hash));
                nRelayedTransactions++;
                {
                    while (!vRelayExpiration.empty() && vRelayExpiration.front().first < nNow)
                    {
                        mapRelay.erase(vRelayExpiration.front().second);
                        vRelayExpiration.pop_front();
                    }
                    auto ret = mapRelay.insert(std::make_pair(hash, std::move(txinfo.tx)));
                    if (ret.second) {
                        vRelayExpiration.push_back(std::make_pair(nNow + 15 * 60 * 1000000, ret.first));
                    }
                }
                if (vInv.size() == MAX_INV_SZ) {
                    connman->PushMessage(pto, msgMaker.Make(NetMsgType::INV, vInv));
                    vInv.clear();
                }
                pto->filterInventoryKnown.insert(hash);
            }

20. 如果 `vInv` 集合不空，那么就调用 `PushMessage` 方法，发送 `INV` 消息。

21. 如果区块状态对象的停止下载区块的时间不等于0，并且小于当前时间减去规定的时间，那么设置区块对象为断开，然后返回真。

        nNow = GetTimeMicros();
        if (state.nStallingSince && state.nStallingSince < nNow - 1000000 * BLOCK_STALLING_TIMEOUT) {
            pto->fDisconnect = true;
            return true;
        }

22. 如果区块下载超时，那么设置区块对象为断开，然后返回真。

        if (state.vBlocksInFlight.size() > 0) {
            QueuedBlock &queuedBlock = state.vBlocksInFlight.front();
            int nOtherPeersWithValidatedDownloads = nPeersWithValidatedDownloads - (state.nBlocksInFlightValidHeaders > 0);
            if (nNow > state.nDownloadingSince + consensusParams.nPowTargetSpacing * (BLOCK_DOWNLOAD_TIMEOUT_BASE + BLOCK_DOWNLOAD_TIMEOUT_PER_PEER * nOtherPeersWithValidatedDownloads)) {
                pto->fDisconnect = true;
                return true;
            }
        }

23. 接下来，检查区块头部下载是否超时。

        if (state.fSyncStarted && state.nHeadersSyncTimeout < std::numeric_limits<int64_t>::max()) {
            if (pindexBestHeader->GetBlockTime() <= GetAdjustedTime() - 24*60*60) {
                if (nNow > state.nHeadersSyncTimeout && nSyncStarted == 1 && (nPreferredDownload - state.fPreferredDownload >= 1)) {
                    if (!pto->fWhitelisted) {
                        LogPrintf("Timeout downloading headers from peer=%d, disconnecting\n", pto->GetId());
                        pto->fDisconnect = true;
                        return true;
                    } else {
                        state.fSyncStarted = false;
                        nSyncStarted--;
                        state.nHeadersSyncTimeout = 0;
                    }
                }
            } else {
                state.nHeadersSyncTimeout = std::numeric_limits<int64_t>::max();
            }
        }

24. 获取所有节点请求的数据，包括区块与非区块，并放在 `vGetData` 集合中，然后调用 `PushMessage` 方法，发送给远程节点。

        std::vector<CInv> vGetData;
        if (!pto->fClient && ((fFetch && !pto->m_limited_node) || !IsInitialBlockDownload()) && state.nBlocksInFlight < MAX_BLOCKS_IN_TRANSIT_PER_PEER) {
            std::vector<const CBlockIndex*> vToDownload;
            NodeId staller = -1;
            FindNextBlocksToDownload(pto->GetId(), MAX_BLOCKS_IN_TRANSIT_PER_PEER - state.nBlocksInFlight, vToDownload, staller, consensusParams);
            for (const CBlockIndex *pindex : vToDownload) {
                uint32_t nFetchFlags = GetFetchFlags(pto);
                vGetData.push_back(CInv(MSG_BLOCK | nFetchFlags, pindex->GetBlockHash()));
                MarkBlockAsInFlight(pto->GetId(), pindex->GetBlockHash(), pindex);
            }
            if (state.nBlocksInFlight == 0 && staller != -1) {
                if (State(staller)->nStallingSince == 0) {
                    State(staller)->nStallingSince = nNow;
                }
            }
        }
        while (!pto->mapAskFor.empty() && (*pto->mapAskFor.begin()).first <= nNow)
        {
            const CInv& inv = (*pto->mapAskFor.begin()).second;
            if (!AlreadyHave(inv))
            {
                LogPrint(BCLog::NET, "Requesting %s peer=%d\n", inv.ToString(), pto->GetId());
                vGetData.push_back(inv);
                if (vGetData.size() >= 1000)
                {
                    connman->PushMessage(pto, msgMaker.Make(NetMsgType::GETDATA, vGetData));
                    vGetData.clear();
                }
            } else {
                pto->setAskFor.erase(inv.hash);
            }
            pto->mapAskFor.erase(pto->mapAskFor.begin());
        }
        if (!vGetData.empty())
            connman->PushMessage(pto, msgMaker.Make(NetMsgType::GETDATA, vGetData));
