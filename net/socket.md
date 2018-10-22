#   P2P 网络建立之套接字处理

P2P 网络的建立是在系统启动的第 12 步，最后时刻调用 `CConnman::Start` 方法开始的。

本部分内容在 `net.cpp`、`net_processing.cpp` 等文件中。


## ThreadSocketHandler

这个线程主要用来处理套接字的读取和发送，调用系统提供的相关相关底层函数进行处理，把收到的消息转化成 `CNetMessage` 类型的对象，并保存到节点的 `vRecvMsg` 集合中，把待发送的消息从 `vSendMsg` 集合中取出来进行发送。

线程定义在 `net.cpp` 文件的 1155 行。下面我们开始进行具体的解读。

这个方法的主体是一个 while 循环，只要 interruptNet 变量为空，就一直循环。

1.  如果布尔变量 fNetworkActive 为假，则断开所有已经连接的接点。

    遍历所有的节点列表，把没有断开的节点设置为断开连接。

        if (!fNetworkActive) {
            // Disconnect any connected nodes
            for (CNode* pnode : vNodes) {
                if (!pnode->fDisconnect) {
                    LogPrint(BCLog::NET, "Network not active, dropping peer=%d\n", pnode->GetId());
                    pnode->fDisconnect = true;
                }
            }
        }

2.  接下来，断开所有未使用的节点。

    遍历所有的节点列表，如果节点已经断开连接，则进行下面的处理：

    -   首先，从 vNodes 集合中删除当前节点。

    -   然后，调用 `CloseSocketDisconnect` 方法，关闭当前节点的套接字并进行清理。

        方法内部设置 `fDisconnect` 变量为真；如果当前套接字有效，则调用 `CloseSocket` 方法，关闭套接字。`CloseSocket` 方法针对 win32 系统调用 `closesocket` 方法，别的系统调用 `close` 来关闭套接字，然后设置套接字为 `INVALID_SOCKET`。

        具体方法如下所示：

            void CNode::CloseSocketDisconnect()
            {
                fDisconnect = true;
                LOCK(cs_hSocket);
                if (hSocket != INVALID_SOCKET)
                {
                    LogPrint(BCLog::NET, "disconnecting peer=%d\n", id);
                    CloseSocket(hSocket);
                }
            }

    -   再然后，调用 `Release` 方法，把节点的引用数量 `nRefCount` 减一。

    -   最后，把当前节点放入 vNodesDisconnected 集合中。

3.  删除所有已断开连接的节点。

    遍历所有已经断开连接的节点的集合 `vNodesDisconnected`，如果当前节点的引用数量 `nRefCount` 小于等于0，进行下面的处理：

    -   调用宏 `TRY_LOCK` 获取 `cs_inventory` 锁。如果可以获取，则获取 `cs_vSend` 锁。如果可以获取，则设置变量 `fDelete` 为真。

    -   如果变量 `fDelete` 为真，则从 `vNodesDisconnected` 集合中移除当前的节点，然后调用 `DeleteNode` 方法，删除节点。

        下面讲解下 `DeleteNode` 方法。方法内部首先设置变量 `fUpdateConnectionTime` 为真，然后调用消息处理接口的 `NetEventsInterface` 的 `FinalizeNode` 方法，最终处理节点。如果变量 `fUpdateConnectionTime` 在 `FinalizeNode` 方法中被设置为真，则调用地址管理器对象的 `Connected` 方法，进行处理。最后，删除当前节点。

        代码如下：

            void CConnman::DeleteNode(CNode* pnode)
            {
                assert(pnode);
                bool fUpdateConnectionTime = false;
                m_msgproc->FinalizeNode(pnode->GetId(), fUpdateConnectionTime);
                if(fUpdateConnectionTime) {
                    addrman.Connected(pnode->addr);
                }
                delete pnode;
            }

        `FinalizeNode` 方法是一个纯虚函数，具体由 `net_processing.cpp` 文件中的 `PeerLogicValidation` 对象的 `FinalizeNode` 方法实现。下面，我们来看这个函数的处理。

        -   设置参数 `fUpdateConnectionTime` 默认为真。

        -   调用 `State` 方法，获取节点状态对象的指针。

            方法内部根据节点 ID，从 `mapNodeState` 集合中找到并返回对应的节点状态对象。如果不存在，则返回空指针。

        -   如果我们已经在这个对等节点上同步了区块头部，即节点状态对象的 `fSyncStarted` 属性为真，则设置变量 `nSyncStarted` 减一。

        -   如果对等节点的不良积分即节点状态对象的 `nMisbehavior` 属性等于0，且对等节点已经建立了完整的连接即节点状态对象的 `fCurrentlyConnected` 属性为真，则设置变量 `fUpdateConnectionTime` 为真。

        -   遍历状态对象的 `vBlocksInFlight` 集合，并删除所有的条目。

        -   调用 `EraseOrphansFor` 方法，删除与这个对等节点相关联的孤儿交易。

            方法内部遍历 `mapOrphanTransactions` 集合，如果当前孤儿交易的来自于指定的对等节点，则调用 `EraseOrphanTx` 方法进行删除。

            后者根据交易的哈希ID，查找 `mapOrphanTransactions` 集合对应的孤儿交易。如果找不到，则返回。如果找到则遍历交易的所有输入，如果当前输入指向的父输出不在 `mapOrphanTransactionsByPrev` 集合中，则处理下一个；否则，在返回的集合中移除指定的孤儿交易，如果指向的父输出现在为空，则从 `mapOrphanTransactionsByPrev` 集合中删除指向的父输出。从 `mapOrphanTransactions` 删除指定的孤儿交易。

                int static EraseOrphanTx(uint256 hash) EXCLUSIVE_LOCKS_REQUIRED(g_cs_orphans)
                {
                    std::map<uint256, COrphanTx>::iterator it = mapOrphanTransactions.find(hash);
                    if (it == mapOrphanTransactions.end())
                        return 0;
                    for (const CTxIn& txin : it->second.tx->vin)
                    {
                        std::set<std::map<uint256, COrphanTx>::iterator, IteratorComparator>

                        auto itPrev = mapOrphanTransactionsByPrev.find(txin.prevout);
                        if (itPrev == mapOrphanTransactionsByPrev.end())
                            continue;
                        itPrev->second.erase(it);
                        if (itPrev->second.empty())
                            mapOrphanTransactionsByPrev.erase(itPrev);
                    }
                    mapOrphanTransactions.erase(it);
                    return 1;
                }

        -   把优先下载区块的对等节点变量`nPreferredDownload` 减去状态对象的对应的 `fPreferredDownload` 属性。这个属性是一个布尔值，表示节点是否为优先下载区块。这里利用了 C++ 布尔值自动转换为整数值，真值转换为1，假值转换为0.

        -   处理正在下载区块的节点数量 `nPeersWithValidatedDownloads` 变量。如果节点状态对象的 `nBlocksInFlightValidHeaders` 属性不等于0，则正在下载区块的节点数量减去1，否则减去0。

        -   接下来处理 `g_outbound_peers_with_protect_from_disconnect` 变量，这个变量代表了出站的对等节点数量。代码比较简单，不解释。

        -   从节点状态集合 `mapNodeState` 中删除指定的节点ID。

4.  如果节点数量不等于 `nPrevNodeCount`，则把 `nPrevNodeCount` 设置为当前的节点数量。

        {
            LOCK(cs_vNodes);
            vNodesSize = vNodes.size();
        }
        if(vNodesSize != nPrevNodeCount) {
            nPrevNodeCount = vNodesSize;
            if(clientInterface)
                clientInterface->NotifyNumConnectionsChanged(nPrevNodeCount);
        }

5.  接下来检查哪个套接字有数据要接收。

    生成相关的时间变量和 3 个 fd_set 集合来处理接收数据、发送数据及数据错误，然后将集合初始化为空集。
        
        struct timeval timeout;
        timeout.tv_sec  = 0;
        timeout.tv_usec = 50000; // frequency to poll pnode->vSend

        fd_set fdsetRecv;
        fd_set fdsetSend;
        fd_set fdsetError;
        FD_ZERO(&fdsetRecv);
        FD_ZERO(&fdsetSend);
        FD_ZERO(&fdsetError);

    遍历当前处于监听状态的套接字 `vhListenSocket` 集合，调用 `FD_SET` 方法，把当前套接字保存在 `fdsetRecv` 集合中，然后调用标准库的 `max` 方法保存最大的套接字，设置变量 `have_fds` 为真。

        for (const ListenSocket& hListenSocket : vhListenSocket) {
            FD_SET(hListenSocket.socket, &fdsetRecv);
            hSocketMax = std::max(hSocketMax, hListenSocket.socket);
            have_fds = true;
        }

    遍历所有的节点，设置相关的变量。

        for (CNode* pnode : vNodes)
        {
            bool select_recv = !pnode->fPauseRecv;
            bool select_send;
            {
                LOCK(pnode->cs_vSend);
                select_send = !pnode->vSendMsg.empty();
            }

            LOCK(pnode->cs_hSocket);
            if (pnode->hSocket == INVALID_SOCKET)
                continue;

            FD_SET(pnode->hSocket, &fdsetError);
            hSocketMax = std::max(hSocketMax, pnode->hSocket);
            have_fds = true;

            if (select_send) {
                FD_SET(pnode->hSocket, &fdsetSend);
                continue;
            }
            if (select_recv) {
                FD_SET(pnode->hSocket, &fdsetRecv);
            }
        }
    
    调用 `select` 方法进行连接。

        int nSelect = select(have_fds ? hSocketMax + 1 : 0,&fdsetRecv, &fdsetSend, &fdsetError, &timeout);

    接下来，检查 `select` 调用是否出错。

        if (nSelect == SOCKET_ERROR)
        {
            if (have_fds)
            {
                int nErr = WSAGetLastError();
                LogPrintf("socket select error %s\n", NetworkErrorString(nErr));
                for (unsigned int i = 0; i <= hSocketMax; i++)
                    FD_SET(i, &fdsetRecv);
            }
            FD_ZERO(&fdsetSend);
            FD_ZERO(&fdsetError);
            if (!interruptNet.sleep_for(std::chrono::milliseconds(timeout.tv_usec/1000)))
                return;
        }

6.  接下来，接收新的连接。

    遍历所有监听的套接字，如果当前套接字有效，并且套接字在在接收数据的集合中，即新的连接请求进来，则调用 `AcceptConnection` 方法进行处理。

        for (const ListenSocket& hListenSocket : vhListenSocket)
        {
            if (hListenSocket.socket != INVALID_SOCKET && FD_ISSET(hListenSocket.socket, &fdsetRecv))
            {
                AcceptConnection(hListenSocket);
            }
        }

    `AcceptConnection` 方法处理远程对等节点的连接逻辑，具体如下：

    -   调用 `accept` 方法，接受客户端连接。

        SOCKET hSocket = accept(hListenSocket.socket, (struct sockaddr*)&sockaddr, &len);

    -   计算最大的入站连接数

            int nMaxInbound = nMaxConnections - (nMaxOutbound + nMaxFeeler);

    -   如果连接成功，那么调用地址对象的 `SetSockAddr` 方法来保存保存对等节点的地址。

            if (hSocket != INVALID_SOCKET) {
                if (!addr.SetSockAddr((const struct sockaddr*)&sockaddr)) {
                    LogPrintf("Warning: Unknown socket family\n");
                }
            }

    -   遍历对等节点列表，如果当前对等节点属于入站类型，则变量 nInbound 加 1。

            {
                LOCK(cs_vNodes);
                for (const CNode* pnode : vNodes) {
                    if (pnode->fInbound) nInbound++;
                }
            }

    -   如果连接不成功，则直接退出。

            if (hSocket == INVALID_SOCKET)
            {
                int nErr = WSAGetLastError();
                if (nErr != WSAEWOULDBLOCK)
                    LogPrintf("socket error accept failed: %s\n", NetworkErrorString(nErr));
                return;
            }

    -   如果网络是不活跃的，则关闭套接字并返回。

            if (!fNetworkActive) {
                LogPrintf("connection from %s dropped: not accepting new connections\n", addr.ToString());
                CloseSocket(hSocket);
                return;
            }

    -   如果是不可连接的，则关闭套接字并返回。

            if (!IsSelectableSocket(hSocket))
            {
                LogPrintf("connection from %s dropped: non-selectable socket\n", addr.ToString());
                CloseSocket(hSocket);
                return;
            }

        `IsSelectableSocket` 方法是一个内联方法，如果 Win32 则直接返回真，否则如果 `select` 函数返回值小于 `FD_SETSIZE` 则返回真，否则返回假。

    -   如果入站数量已经达到最大的入站数量，则调用 `AttemptToEvictConnection` 方法，找到要退出的连接。如果找不到则关闭套接字并返回。

            if (nInbound >= nMaxInbound)
            {
                if (!AttemptToEvictConnection()) {
                    LogPrint(BCLog::NET, "failed to find an eviction candidate - connection dropped (full)\n");
                    CloseSocket(hSocket);
                    return;
                }
            }

    -   生成节点对象，并进行相关设置，然后加入节点集合中 `vNodes`。

            NodeId id = GetNewNodeId();
            uint64_t nonce = GetDeterministicRandomizer(RANDOMIZER_ID_LOCALHOSTNONCE).Write(id).Finalize();
            CAddress addr_bind = GetBindAddress(hSocket);

            CNode* pnode = new CNode(id, nLocalServices, GetBestHeight(), hSocket, addr, CalculateKeyedNetGroup(addr), nonce, addr_bind, "", true);
            pnode->AddRef();
            pnode->fWhitelisted = whitelisted;
            m_msgproc->InitializeNode(pnode);

            LogPrint(BCLog::NET, "connection from %s accepted\n", addr.ToString());

            {
                LOCK(cs_vNodes);
                vNodes.push_back(pnode);
            }

7.  处理节点的引用数量

        std::vector<CNode*> vNodesCopy;
        {
            LOCK(cs_vNodes);
            vNodesCopy = vNodes;
            for (CNode* pnode : vNodesCopy)
                pnode->AddRef();
        }

    `AddRef` 方法会把 `nRefCount` 加1。

8.  遍历所有的节点进行收发信息处理。

    -   判断当前节点是否在读写、错误集合中。

            bool recvSet = false;
            bool sendSet = false;
            bool errorSet = false;
            {
                LOCK(pnode->cs_hSocket);
                if (pnode->hSocket == INVALID_SOCKET)
                    continue;
                recvSet = FD_ISSET(pnode->hSocket, &fdsetRecv);
                sendSet = FD_ISSET(pnode->hSocket, &fdsetSend);
                errorSet = FD_ISSET(pnode->hSocket, &fdsetError);
            }

    -   如果当前节点在读取集合或错误集合中，则进行下面的处理：

        调用 `recv` 方法，读取数据。

            char pchBuf[0x10000];
            int nBytes = 0;
            {
                LOCK(pnode->cs_hSocket);
                if (pnode->hSocket == INVALID_SOCKET)
                    continue;
                nBytes = recv(pnode->hSocket, pchBuf, sizeof(pchBuf), MSG_DONTWAIT);
            }

         如果读取的数量大于0，则：调用 `ReceiveMsgBytes` 方法，从缓冲区中读取给定数量的数据，并生成 `CNetMessage` 对象。如果出错，则调用 `CloseSocketDisconnect` 方法，关闭套接字并断开连接。

         如果读取的数量等于0，即远程节点已经关闭，则：调用 `CloseSocketDisconnect` 方法，关闭套接字并断开连接。

         如果读取的数量小于0，即读取过程中出错，则：调用 `CloseSocketDisconnect` 方法，关闭套接字并断开连接。

         具体代码如下：

            if (recvSet || errorSet)
            {
                // typical socket buffer is 8K-64K
                char pchBuf[0x10000];
                int nBytes = 0;
                {
                    LOCK(pnode->cs_hSocket);
                    if (pnode->hSocket == INVALID_SOCKET)
                        continue;
                    nBytes = recv(pnode->hSocket, pchBuf, sizeof(pchBuf), MSG_DONTWAIT);
                }
                if (nBytes > 0)
                {
                    bool notify = false;
                    if (!pnode->ReceiveMsgBytes(pchBuf, nBytes, notify))
                        pnode->CloseSocketDisconnect();
                    RecordBytesRecv(nBytes);
                    if (notify) {
                        size_t nSizeAdded = 0;
                        auto it(pnode->vRecvMsg.begin());
                        for (; it != pnode->vRecvMsg.end(); ++it) {
                            if (!it->complete())
                                break;
                            nSizeAdded += it->vRecv.size() + CMessageHeader::HEADER_SIZE;
                        }
                        {
                            LOCK(pnode->cs_vProcessMsg);
                            pnode->vProcessMsg.splice(pnode->vProcessMsg.end(), pnode->vRecvMsg, pnode->vRecvMsg.begin(), it);
                            pnode->nProcessQueueSize += nSizeAdded;
                            pnode->fPauseRecv = pnode->nProcessQueueSize > nReceiveFloodSize;
                        }
                        WakeMessageHandler();
                    }
                }
                else if (nBytes == 0)
                {
                    // socket closed gracefully
                    if (!pnode->fDisconnect) {
                        LogPrint(BCLog::NET, "socket closed\n");
                    }
                    pnode->CloseSocketDisconnect();
                }
                else if (nBytes < 0)
                {
                    // error
                    int nErr = WSAGetLastError();
                    if (nErr != WSAEWOULDBLOCK && nErr != WSAEMSGSIZE && nErr != WSAEINTR && nErr != WSAEINPROGRESS)
                    {
                        if (!pnode->fDisconnect)
                            LogPrintf("socket recv error %s\n", NetworkErrorString(nErr));
                        pnode->CloseSocketDisconnect();
                    }
                }
            }

    -   如果当前节点在发送集合中，则进行如下处理。

            if (sendSet)
            {
                LOCK(pnode->cs_vSend);
                size_t nBytes = SocketSendData(pnode);
                if (nBytes) {
                    RecordBytesSent(nBytes);
                }
            }

        `SocketSendData` 方法主要逻辑是遍历节点的发送消息集合 `vSendMsg`，然后调用 `send` 方法发送每一个消息，并针对发送正确与否进行处理；同时从发送消息集合 `vSendMsg` 对应的消息。

            size_t CConnman::SocketSendData(CNode *pnode) const
            {
                auto it = pnode->vSendMsg.begin();
                size_t nSentSize = 0;

                while (it != pnode->vSendMsg.end()) {
                    const auto &data = *it;
                    assert(data.size() > pnode->nSendOffset);
                    int nBytes = 0;
                    {
                        LOCK(pnode->cs_hSocket);
                        if (pnode->hSocket == INVALID_SOCKET)
                            break;
                        nBytes = send(pnode->hSocket, reinterpret_cast<const char*>(data.data()) + pnode->nSendOffset, data.size() - pnode->nSendOffset, MSG_NOSIGNAL | MSG_DONTWAIT);
                    }
                    if (nBytes > 0) {
                        pnode->nLastSend = GetSystemTimeInSeconds();
                        pnode->nSendBytes += nBytes;
                        pnode->nSendOffset += nBytes;
                        nSentSize += nBytes;
                        if (pnode->nSendOffset == data.size()) {
                            pnode->nSendOffset = 0;
                            pnode->nSendSize -= data.size();
                            pnode->fPauseSend = pnode->nSendSize > nSendBufferMaxSize;
                            it++;
                        } else {
                            // could not send full message; stop sending more
                            break;
                        }
                    } else {
                        if (nBytes < 0) {
                            // error
                            int nErr = WSAGetLastError();
                            if (nErr != WSAEWOULDBLOCK && nErr != WSAEMSGSIZE && nErr != WSAEINTR && nErr != WSAEINPROGRESS)
                            {
                                LogPrintf("socket send error %s\n", NetworkErrorString(nErr));
                                pnode->CloseSocketDisconnect();
                            }
                        }
                        // couldn't send anything at all
                        break;
                    }
                }

                if (it == pnode->vSendMsg.end()) {
                    assert(pnode->nSendOffset == 0);
                    assert(pnode->nSendSize == 0);
                }
                pnode->vSendMsg.erase(pnode->vSendMsg.begin(), it);
                return nSentSize;
            }

9.  接下来对节点的活跃性进行检查。

        int64_t nTime = GetSystemTimeInSeconds();
        if (nTime - pnode->nTimeConnected > 60)
        {
            if (pnode->nLastRecv == 0 || pnode->nLastSend == 0)
            {
                LogPrint(BCLog::NET, "socket no message in first 60 seconds, %d %d from %d\n", pnode->nLastRecv != 0, pnode->nLastSend != 0, pnode->GetId());
                pnode->fDisconnect = true;
            }
            else if (nTime - pnode->nLastSend > TIMEOUT_INTERVAL)
            {
                LogPrintf("socket sending timeout: %is\n", nTime - pnode->nLastSend);
                pnode->fDisconnect = true;
            }
            else if (nTime - pnode->nLastRecv > (pnode->nVersion > BIP0031_VERSION ? TIMEOUT_INTERVAL : 90*60))
            {
                LogPrintf("socket receive timeout: %is\n", nTime - pnode->nLastRecv);
                pnode->fDisconnect = true;
            }
            else if (pnode->nPingNonceSent && pnode->nPingUsecStart + TIMEOUT_INTERVAL * 1000000 < GetTimeMicros())
            {
                LogPrintf("ping timeout: %fs\n", 0.000001 * (GetTimeMicros() - pnode->nPingUsecStart));
                pnode->fDisconnect = true;
            }
            else if (!pnode->fSuccessfullyConnected)
            {
                LogPrint(BCLog::NET, "version handshake timeout from %d\n", pnode->GetId());
                pnode->fDisconnect = true;
            }
        }
    }

10. 遍历所有节点，减少引用数。

        {
            LOCK(cs_vNodes);
            for (CNode* pnode : vNodesCopy)
                pnode->Release();
        }

    `Release` 方法把 `nRefCount` 参数减1。

