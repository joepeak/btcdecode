#   P2P 网络建立之消息处理中篇

P2P 网络的建立是在系统启动的第 12 步，最后时刻调用 `CConnman::Start` 方法开始的。

恭喜你越来越接近比特币的核心了，在中[上篇](https://blog.csdn.net/xiyuan688/article/details/83344174)，我们主要讲解了比特币的消息处理线程，接下来，在下篇中，将以具体的比特币消息即比特币协义分析为主。针对比特币的协义，为了从逻辑上进行理解，我们并没有完全按照代码的顺序，而是按照某个具体的消息的 `请求----响应` 模式来进行分析。

下面我们来看比特币协义相关的代码。

##  1、节点握手处理

### 1.1、接收 `version` 消息

节点作为服务器，处理客户端节点发送的版本请求。

版本消息是每个对等节点都要发送的消息，并且是最先发送、只能发送一次的消息，对等节点双方都要发送这个消息和下面的确认消息，只有双方都发送过版本消息，并且收到确认消息，对等节点间才可以进行后续消息发送。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 1621 行。具体处理如下：

1.  检查对等节点的版本字段是否已经设置，如果已经设置，即远程对等节点已经发送过版本消息，那么：在开启 BIP 161 情况下发送拒绝消息；对远程对待节点进行处罚。

        if (pfrom->nVersion != 0)
        {
            if (enable_bip61) {
                connman->PushMessage(pfrom, CNetMsgMaker(INIT_PROTO_VERSION).Make(NetMsgType::REJECT, strCommand, REJECT_DUPLICATE, std::string("Duplicate version message")));
            }
            LOCK(cs_main);
            Misbehaving(pfrom->GetId(), 1);
            return false;
        }

    `Misbehaving` 方法进行处理，具体处理如下：

    -   检查是否要增加节点的不良积分，如果不是，即增加的积分数量为 0，则直接退出。

    -   取得节点的状态对象。如果不存在，则直接退出。

    -   把节点的状态对象的不良积分加上要增加的不良积分。

    -   比较节点的不良积分与默认的或用户通过 `-banscore` 参数指定的不良积分进行比较。如果在增加当前不良积分后大于等于设置的不良积分，并且增加之前小于设置的不良积分，那么设置状态对象为应该禁止，即设置 `fShouldBan` 属性为真。

    这个方法的代码如下：

        void Misbehaving(NodeId pnode, int howmuch, const std::string& message) EXCLUSIVE_LOCKS_REQUIRED(cs_main)
        {
            if (howmuch == 0)
                return;

            CNodeState *state = State(pnode);
            if (state == nullptr)
                return;

            state->nMisbehavior += howmuch;
            int banscore = gArgs.GetArg("-banscore", DEFAULT_BANSCORE_THRESHOLD);
            std::string message_prefixed = message.empty() ? "" : (": " + message);
            if (state->nMisbehavior >= banscore && state->nMisbehavior - howmuch < banscore)
            {
                LogPrint(BCLog::NET, "%s: %s peer=%d (%d -> %d) BAN THRESHOLD EXCEEDED%s\n", __func__, state->name, pnode, state->nMisbehavior-howmuch, state->nMisbehavior, message_prefixed);
                state->fShouldBan = true;
            } else
                LogPrint(BCLog::NET, "%s: %s peer=%d (%d -> %d)%s\n", __func__, state->name, pnode, state->nMisbehavior-howmuch, state->nMisbehavior, message_prefixed);
        }

2.  从输入流中取得远程对等节点发送的版本信息、支持的服务信息、发送时间、显示地址。

        vRecv >> nVersion >> nServiceInt >> nTime >> addrMe;
        nSendVersion = std::min(nVersion, PROTOCOL_VERSION);
        nServices = ServiceFlags(nServiceInt);

3.  如果对等节点是出站的，调用 `CConnman` 对象的 `SetServices` 方法，设置对等节点所支持的服务。

        if (!pfrom->fInbound)
        {
            connman->SetServices(pfrom->addr, nServices);
        }

    方法内部最终会获取节点的地址信息对象，然后设置其支持的服务属性。

4.  如果对等节点是出站的，且不是临时的试探者，且不是手动指定的，且与本地服务不匹配，那么：在开启 BIP 161 情况下发送拒绝消息；然后设置远程对待节点为断开；然后返回假。

        if (!pfrom->fInbound && !pfrom->fFeeler && !pfrom->m_manual_connection && !HasAllDesirableServiceFlags(nServices))
        {
            LogPrint(BCLog::NET, "peer=%d does not offer the expected services (%08x offered, %08x expected); disconnecting\n", pfrom->GetId(), nServices, GetDesirableServiceFlags(nServices));
            if (enable_bip61) {
                connman->PushMessage(pfrom, CNetMsgMaker(INIT_PROTO_VERSION).Make(NetMsgType::REJECT, strCommand, REJECT_NONSTANDARD,strprintf("Expected to offer services %08x", GetDesirableServiceFlags(nServices))));
            }
            pfrom->fDisconnect = true;
            return false;
        }

5.  如果发送的版本的小于协义规定的最小版本 `MIN_PEER_PROTO_VERSION`，那么：在开启 BIP 161 情况下发送拒绝消息；然后设置远程对待节点为断开；然后返回假。

        if (nVersion < MIN_PEER_PROTO_VERSION) {
            // disconnect from peers older than this proto version
            LogPrint(BCLog::NET, "peer=%d using obsolete version %i; disconnecting\n", pfrom->GetId(), nVersion);
            if (enable_bip61) {
                connman->PushMessage(pfrom, CNetMsgMaker(INIT_PROTO_VERSION).Make(NetMsgType::REJECT, strCommand, REJECT_OBSOLETE,
                                   strprintf("Version must be %d or greater", MIN_PEER_PROTO_VERSION)));
            }
            pfrom->fDisconnect = true;
            return false;
        }

6.  如果输入流不为空，则从流中依次取得 addrFrom、nNonce、strSubVer（客户端字符串）、nStartingHeight（客户端区块链的高度）、fRelay等信息。

        if (!vRecv.empty())
            vRecv >> addrFrom >> nNonce;
        if (!vRecv.empty()) {
            vRecv >> LIMITED_STRING(strSubVer, MAX_SUBVERSION_LENGTH);
            cleanSubVer = SanitizeString(strSubVer);
        }
        if (!vRecv.empty()) {
            vRecv >> nStartingHeight;
        }
        if (!vRecv.empty())
            vRecv >> fRelay;

7.  如果对等节点节点是入站节点，且连接到自身，那么设置远程对待节点为断开，并返回真。

        if (pfrom->fInbound && !connman->CheckIncomingNonce(nNonce))
        {
            LogPrintf("connected to self at %s, disconnecting\n", pfrom->addr.ToString());
            pfrom->fDisconnect = true;
            return true;
        }

8.  如果对等节点是入站节点，且其地址是可路由的，那么调用 `SeenLocal` 方法进行处理。

        if (pfrom->fInbound && addrMe.IsRoutable())
        {
            SeenLocal(addrMe);
        }

    在 `SeenLocal` 方法中，如果这个地址在 `mapLocalHost` 集合中存在，那么设置其对应的 `LocalServiceInfo` 对象的 `nScore` 加1。如晨不存在，则直接返回真。

        bool SeenLocal(const CService& addr)
        {
            {
                LOCK(cs_mapLocalHost);
                if (mapLocalHost.count(addr) == 0)
                    return false;
                mapLocalHost[addr].nScore++;
            }
            return true;
        }

9.  **如果是对等节点是入站节点，则调用 `PushNodeVersion` 方法，发送自身的版本信息给远程对等节点**。

    节点在收到远程对待节点发送来的版本消息，并且经过检查没问题之后，自身发送一个版本消息给对远程对待节点。

10.  **调用 `CConnman` 对象的 `PushMessage` 方法，发送版本确认包**。

    因为当前的 `version` 消息，是别的节点请求我们的，当我们允许其连接时，发送版本确认包。注意，只有在双方都发送版本确认包之后，双方才可以互相发送消息。

11. 设置对等节点的服务属性、保存地址、对等节点运行的客户端、对等节点区块链的高度、版本等。如果对等节点隔离见证服务，则设置对等节点对应的状态对象的相关属性为真。

        pfrom->nServices = nServices;
        pfrom->SetAddrLocal(addrMe);
        {
            LOCK(pfrom->cs_SubVer);
            pfrom->strSubVer = strSubVer;
            pfrom->cleanSubVer = cleanSubVer;
        }
        pfrom->nStartingHeight = nStartingHeight;
        pfrom->fClient = (!(nServices & NODE_NETWORK) && !(nServices & NODE_NETWORK_LIMITED));
        pfrom->m_limited_node = (!(nServices & NODE_NETWORK) && (nServices & NODE_NETWORK_LIMITED));
        pfrom->fRelayTxes = fRelay;
        pfrom->SetSendVersion(nSendVersion);
        pfrom->nVersion = nVersion;

12. 调用 `UpdatePreferredDownload` 方法，将对等节点设为可能的首先下载节点。

    如果节点是出站的或者在白名单中，并且可以提供区块服务，并且 `fOneShot` 属性为假，即可作为首选下载节点。

13. 如果对等节点不是入站节点，进行如下处理。

    -   如果对等节点不是孤立的，并且不需要进行IBD下载（调用 `IsInitialBlockDownload` 函数进行判断，通常第一次启动或在常时间离线，比如24小时，有大师区块需要下载时，本方法返回真），那么：

        -   调用 `GetLocalAddress` 方法，获取对该对等节点来说是最佳的地址。

        -   如果找到的地址是可路由的，那么调用对等节点的 `PushAddress` 方法，把找到的地址保存在 `vAddrToSend` 集合中。

        -   否则，调用 `IsPeerAddrLocalGood` 测试远程对等节点看到的我们的外部IP是否可以路由。如果可以路由，那么调用对等节点的 `PushAddress` 方法，把地址保存在 `vAddrToSend` 集合中。

        以上代码如下：

            if (fListen && !IsInitialBlockDownload())
            {
                CAddress addr = GetLocalAddress(&pfrom->addr, pfrom->GetLocalServices());
                FastRandomContext insecure_rand;
                if (addr.IsRoutable())
                {
                    LogPrint(BCLog::NET, "ProcessMessages: advertising address %s\n", addr.ToString());
                    pfrom->PushAddress(addr, insecure_rand);
                } else if (IsPeerAddrLocalGood(pfrom)) {
                    addr.SetIP(addrMe);
                    LogPrint(BCLog::NET, "ProcessMessages: advertising address %s\n", addr.ToString());
                    pfrom->PushAddress(addr, insecure_rand);
                }
            }

    -   如果需要，比如：本地保存的远程地址少于 1000个，那么调**用 `PushMessage` 方法，请求远程节点发送更多的地址，即发送`getaddr` 消息**。然后把请求地址的标志设置为真。

            if (pfrom->fOneShot || pfrom->nVersion >= CADDR_TIME_VERSION || connman->GetAddressCount() < 1000)
            {
                connman->PushMessage(pfrom, CNetMsgMaker(nSendVersion).Make(NetMsgType::GETADDR));
                pfrom->fGetAddr = true;
            }

    -   调用 `MarkAddressGood` 方法，保存远程对等节点，表明它是可访问的。

            connman->MarkAddressGood(pfrom->addr);

14. 如果远程对等节点的版本小于 70012，则发送一个 `alert` 消息。

        if (pfrom->nVersion <= 70012) {
            CDataStream finalAlert(ParseHex("60010000000000000000000000ffffff7f00000000ffffff7ffeffff7f01ffffff7f00000000ffffff7f00ffffff7f002f555247454e543a20416c657274206b657920636f6d70726f6d697365642c2075706772616465207265717569726564004630440220653febd6410f470f6bae11cad19c48413becb1ac2c17f908fd0fd53bdc3abd5202206d0e9c96fe88d4a0f01ed9dedae2b6f9e00da94cad0fecaae66ecf689bf71b50"), SER_NETWORK, PROTOCOL_VERSION);
            connman->PushMessage(pfrom, CNetMsgMaker(nSendVersion).Make("alert", finalAlert));
        }

15. 如果节点是临时引导节点，则断开节点，即设置节点的断开属性为真。

        if (pfrom->fFeeler) {
            assert(pfrom->fInbound == false);
            pfrom->fDisconnect = true;
        }

16. 版本消息处理完成，返回真。


### 1.2、接收 `verack` 消息

节点作为客户端，处理服务器节点发送的版本响应消息，即版本确认消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 1805 行。具体处理过程如下：

1.  设置接收到的版本确认消息中的版本号。

        pfrom->SetRecvVersion(std::min(pfrom->nVersion.load(), PROTOCOL_VERSION));

2.  如果对等节点不是入站节点，设置对等节点的状态对象的当前连接属性为真。

        if (!pfrom->fInbound) {
            LOCK(cs_main);
            State(pfrom->GetId())->fCurrentlyConnected = true;
        }

2.  如果对等节点的版本大于支持使用区块头部来公告区块的最小版本（`SENDHEADERS_VERSION = 70012`），那么：

    **调用 `PushMessage` 方法发送 `sendheaders` 消息，通知远程对等节点我们更愿意通过 `headers` 消息来接收新区块的公告，而不是 `inv` 消息**。

    这样以后当有新区块需要公告时，远程对等就会通过 `headers` 消息把区块头部先发送给我们，当我们再次请求时才会发送完整的区块。

3.  如果对等节点的版本大于支持紧凑区块的最小版本（`SHORT_IDS_BLOCKS_VERSION = 70014`），那么分两种情况处理。

    -   第一种情况，如果同时支持闪电网络，那么给**对等节点发送一个紧凑区块版本为 2 的 `sendcmpct` 消息**。

    -   第二种情况，如果不支持闪电网络，那么给**对等节点发送一个紧凑区块版本为 1 的 `sendcmpct` 消息**。

    无论哪一种情况，远程对等节点以后都会向本节点发送紧凑区块。

    代码如下：

        if (pfrom->nVersion >= SHORT_IDS_BLOCKS_VERSION) {
            bool fAnnounceUsingCMPCTBLOCK = false;
            uint64_t nCMPCTBLOCKVersion = 2;
            if (pfrom->GetLocalServices() & NODE_WITNESS)
                connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::SENDCMPCT, fAnnounceUsingCMPCTBLOCK, nCMPCTBLOCKVersion));
            nCMPCTBLOCKVersion = 1;
            connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::SENDCMPCT, fAnnounceUsingCMPCTBLOCK, nCMPCTBLOCKVersion));
        }

4.  设置对等节点完全成功连接的标志为真，然后返回真。

        pfrom->fSuccessfullyConnected = true;
        return true;

只有在对等节点双方都各自发送版本消息和确认消息之后，双方才真正建立起连接关系，才可以进行后续的交互，比如请求数据消息等。


##  2、保持连接的处理

因为在比特币网络中，任何一个节点都可以随时加入网络，也可以随时离开网络，所以两个连接的节点需要定时互相发送 `ping` 和 `pong` 来确保接点可以连接，如果在特定的时间内没有 `ping` 消息，节点即可认为连接已经断开。

### 2.1、`ping` 消息

这个消息比较简单，不作具体解释，代码如下：

    if (strCommand == NetMsgType::PING) {
        if (pfrom->nVersion > BIP0031_VERSION)
        {
            uint64_t nonce = 0;
            vRecv >> nonce;
            connman->PushMessage(pfrom, msgMaker.Make(NetMsgType::PONG, nonce));
        }
        return true;
    }


### 2.1、`pong` 消息

这个消息也比较简单，不作具体解释，代码如下：

    if (strCommand == NetMsgType::PONG) {
        int64_t pingUsecEnd = nTimeReceived;
        uint64_t nonce = 0;
        size_t nAvail = vRecv.in_avail();
        bool bPingFinished = false;
        std::string sProblem;
        if (nAvail >= sizeof(nonce)) {
            vRecv >> nonce;
            if (pfrom->nPingNonceSent != 0) {
                if (nonce == pfrom->nPingNonceSent) {
                    bPingFinished = true;
                    int64_t pingUsecTime = pingUsecEnd - pfrom->nPingUsecStart;
                    if (pingUsecTime > 0) {
                        pfrom->nPingUsecTime = pingUsecTime;
                        pfrom->nMinPingUsecTime = std::min(pfrom->nMinPingUsecTime.load(), pingUsecTime);
                    } else {
                        sProblem = "Timing mishap";
                    }
                } else {
                    sProblem = "Nonce mismatch";
                    if (nonce == 0) {
                        bPingFinished = true;
                        sProblem = "Nonce zero";
                    }
                }
            } else {
                sProblem = "Unsolicited pong without ping";
            }
        } else {
            bPingFinished = true;
            sProblem = "Short payload";
        }
        if (bPingFinished) {
            pfrom->nPingNonceSent = 0;
        }
        return true;
    }



##  3、获取更多地址的处理

如果对等节点需要更多地址时，会发送 `getaddr` 消息请求远程对等节点发送更多的地址，当远程对等节点收到请求后，会通过发送 `addr` 消息传递更多的地址。

### 3.1、`getaddr` 消息

节点作为服务器，响应客户端节点发送的请求地址消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2728 行。具体处理如下：

1.  如果对等节点不是入站节点，则忽略该请求，并返回。

        if (!pfrom->fInbound) {
            return true;
        }

2.  如果对等节点已发送过请求地址，即远程对等节点重复请求地址，则忽略该请求，并返回。

        if (pfrom->fSentAddr) {
            return true;
        }

3.  设置对等节点已发送过请求地址标志为真。清空对等节点的 `vAddrToSend` 集合。

        pfrom->fSentAddr = true;
        pfrom->vAddrToSend.clear();

5.  调用 `GetAddresses` 方法，返回要发送的地址。

    从地址管理器随机找到 N 个地址，N不能大于最大值 2500，并且这些地址的状态都比较好。

6.  遍历要发送的节点，调用对等节点的 `PushAddress` 方法，把要发送的地址保存到 `vAddrToSend` 向量中。

    **由线程进行定时发送 `addr` 消息**。

7.  返回真。


### 3.2、`addr` 消息

节点作为客户端，响应服务器节点返回的地址。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 1849 行。具体处理如下：

1.  从输入流中取得远程对等节点发送的地址列表保存到 `vAddr` 向量中。

        std::vector<CAddress> vAddr;
        vRecv >> vAddr;

2.  如果远程对等节点的版本小于 31402（在这种版本比较老的情况下，我们只在初始时接收 DNS 种子服务器发送的地址），并且本保存的地址已经超过 1000，则直接返回真。

        if (pfrom->nVersion < CADDR_TIME_VERSION && connman->GetAddressCount() > 1000)
            return true;

3.  如果远程对等节点发送的地址超过 1000，调用 `Misbehaving` 方法，对远程节点进行惩罚，可能导致其被禁止发送。

        if (vAddr.size() > 1000)
        {
            LOCK(cs_main);
            Misbehaving(pfrom->GetId(), 20, strprintf("message addr size() = %u", vAddr.size()));
            return false;
        }

4.  遍历所有的地址列表，进行如下处理：

    -   如果线程被中止，则返回真。

    -   如果代表的节点不支持 `NODE_NETWORK`、`NODE_NETWORK_LIMITED` 两者之一的服务，则处理下一个。

            if (!MayHaveUsefulAddressDB(addr.nServices) && !HasAllDesirableServiceFlags(addr.nServices))
                continue;

    -   设置地址的时间属性

            if (addr.nTime <= 100000000 || addr.nTime > nNow + 10 * 60)
                addr.nTime = nNow - 5 * 24 * 60 * 60;

    -   调用对等节点的 `AddAddressKnown` 方法，把当前地址保存到已知地址 `addrKnown` 中。

    -   如果地址是可路由的，则加入 `vAddrOk` 列表中。

            if (fReachable)
                vAddrOk.push_back(addr);

5.  调用 `CConnman::AddNewAddresses` 方法，保存 `vAddrOk` 列表中的地址。

    `AddNewAddresses` 方法最终把地址列表保存在 `CAddrMan` 对象的 `mapInfo` 属性中。

6.  如果发送的地址数量少于 1000，设置对等节点的获取地址标志为假。以便以后再次获取地址。如果对等节点的 `fOneShot` 属性为真，则设置对等节点的断开连接标志为真。

        if (vAddr.size() < 1000)
            pfrom->fGetAddr = false;
        if (pfrom->fOneShot)
            pfrom->fDisconnect = true;

7.  返回真。



##  4、`sendheaders` 消息

节点作为服务器，响应客户端节点发送的更愿意接收头部而不是区块体的设置消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 1899 行。这个消息的处理比较简单，只把节点对应的状态对象的 `fPreferHeaders` 属性为真。

代码如下：

    if (strCommand == NetMsgType::SENDHEADERS) {
        LOCK(cs_main);
        State(pfrom->GetId())->fPreferHeaders = true;
        return true;
    }



##  5、`sendcmpct` 消息

节点作为服务器，响应客户端节点发送的接收紧凑区块而不是普通区块的设置消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 1905 行。

以 `if (strCommand == NetMsgType::SENDCMPCT) { ` 为开始，具体如下：

1.  从输入流中取得 `fAnnounceUsingCMPCTBLOCK`、`nCMPCTBLOCKVersion` 等参数。

        vRecv >> fAnnounceUsingCMPCTBLOCK >> nCMPCTBLOCKVersion;

2.  如果紧凑区块版本 `nCMPCTBLOCKVersion` 等于 1 ，或者节点可以响应包含隔离见证的区块和交易请求，且 `nCMPCTBLOCKVersion` 等于2，那么进行下面的处理。

        if (nCMPCTBLOCKVersion == 1 || ((pfrom->GetLocalServices() & NODE_WITNESS) && nCMPCTBLOCKVersion == 2)) {
            LOCK(cs_main);
            if (!State(pfrom->GetId())->fProvidesHeaderAndIDs) {
                State(pfrom->GetId())->fProvidesHeaderAndIDs = true;
                State(pfrom->GetId())->fWantsCmpctWitness = nCMPCTBLOCKVersion == 2;
            }
            if (State(pfrom->GetId())->fWantsCmpctWitness == (nCMPCTBLOCKVersion == 2)) // ignore later version announces
                State(pfrom->GetId())->fPreferHeaderAndIDs = fAnnounceUsingCMPCTBLOCK;
            if (!State(pfrom->GetId())->fSupportsDesiredCmpctVersion) {
                if (pfrom->GetLocalServices() & NODE_WITNESS)
                    State(pfrom->GetId())->fSupportsDesiredCmpctVersion = (nCMPCTBLOCKVersion == 2);
                else
                    State(pfrom->GetId())->fSupportsDesiredCmpctVersion = (nCMPCTBLOCKVersion == 1);
            }
        }

3.  返回真。



##  6、`feefilter` 消息

节点作为服务器，响应客户端节点发送的费率过滤设置消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2911 行。

以 `if (strCommand == NetMsgType::FEEFILTER) {` 为开始，这个处理比较简单，代码如下：

    if (strCommand == NetMsgType::FEEFILTER) {
        CAmount newFeeFilter = 0;
        vRecv >> newFeeFilter;
        if (MoneyRange(newFeeFilter)) {
            {
                LOCK(pfrom->cs_feeFilter);
                pfrom->minFeeFilter = newFeeFilter;
            }
            LogPrint(BCLog::NET, "received: feefilter of %s from peer=%d\n", CFeeRate(newFeeFilter).ToString(), pfrom->GetId());
        }
        return true;
    }

其中 `MoneyRange` 方法检查费率参数是否在 0 到 2100 万比特币之间。



##  7、`filterload` 消息

节点作为服务器，响应 SPV 节点发送的布隆过滤设置消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2851 行。

以 ` if (strCommand == NetMsgType::FILTERLOAD) {` 为开始，具体如下：

1.  从输入流中取得过滤器参数。

        CBloomFilter filter;
        vRecv >> filter;

2.  调用布隆过滤器的 `IsWithinSizeConstraints` 方法，检查过滤器的是否超过限制区间。如果超过，则调用 `Misbehaving` 方法，对远程对等节点进行设置，可能导致其被禁止。如果不超过，则：

    -   生成一个新的 `CBloomFilter` 过滤器对象，并设置节点的过滤器属性 `pfilter` 为新生成的对象。

    -   调用节点过滤器的 `UpdateEmptyFull` 方法，重置其内部属性 `vData`。

    -   设置中继交易属性 `fRelayTxes` 为真。

    以上代码如下：

        if (!filter.IsWithinSizeConstraints())
        {
            LOCK(cs_main);
            Misbehaving(pfrom->GetId(), 100);
        }
        else
        {
            LOCK(pfrom->cs_filter);
            pfrom->pfilter.reset(new CBloomFilter(filter));
            pfrom->pfilter->UpdateEmptyFull();
            pfrom->fRelayTxes = true;
        }

3.  返回真。



### 8、`filteradd` 消息

节点作为服务器，响应 SPV 节点发送的增加布隆过滤消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2871 行。

以 ` if (strCommand == NetMsgType::FILTERADD) {` 为开始，具体如下：

1.  从输入流中取得要增加的过滤器。

        std::vector<unsigned char> vData;
        vRecv >> vData;

2.  如果发送的字节数量大于 520，则设置变量 `bad` 为真。如果不大于，则进行下面的判断。

    如果已经发送过 `filterload` 消息，则把新的过滤器保存到过滤器集合中。否则，设置变量 `bad` 为真。

    代码如下：

        bool bad = false;
        if (vData.size() > MAX_SCRIPT_ELEMENT_SIZE) {
            bad = true;
        } else {
            LOCK(pfrom->cs_filter);
            if (pfrom->pfilter) {
                pfrom->pfilter->insert(vData);
            } else {
                bad = true;
            }
        }

3.  如果变量为真，调用 `Misbehaving` 方法，惩罚节点。

4.  返回真。



##  9、`filterclear`

节点作为服务器，响应 SPV 节点发送的增加布隆过滤消息。

代码在 `net_processing.cpp` 文件中的 `ProcessMessage` 方法的 2895 行。

以 ` if (strCommand == NetMsgType::FILTERCLEAR) {` 为开始，这个消息处理比较简单，代码如下，可以自己理解。

    if (strCommand == NetMsgType::FILTERCLEAR) {
        LOCK(pfrom->cs_filter);
        if (pfrom->GetLocalServices() & NODE_BLOOM) {
            pfrom->pfilter.reset(new CBloomFilter());
        }
        pfrom->fRelayTxes = true;
        return true;
    }


##  10、`mempool` 消息

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
