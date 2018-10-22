#   P2P 网络建立之连接对等节点处理

P2P 网络的建立是在系统启动的第 12 步，最后时刻调用 `CConnman::Start` 方法开始的。

本部分内容在 `net.cpp`、`net_processing.cpp` 等文件中。

## ThreadOpenAddedConnections

这个线程的主要作用是生成地址对象，并且调用 `OpenNetworkConnection` 方法，连接到指定地址。

线程定义在 `net.cpp` 文件的 1959 行。下面我们开始进行具体的解读。

线程的主体是一个 `while` 循环。在循环中进行下面的处理。

1.  调用 `GetAddedNodeInfo` 方法，获取所有的节点信息。

    本方法返回所有的节点信息，其中即有已连接的，也有未连接的地址。

    -   首先，生成保存节点信息的容器变量 `ret` 和保存地址字符串的列表对象 `lAddresses`。然后把 `vAddedNodes` 集合中的所有地址拷贝到 `lAddresses` 中。

            std::vector<AddedNodeInfo> ret;

            std::list<std::string> lAddresses(0);
            {
                LOCK(cs_vAddedNodes);
                ret.reserve(vAddedNodes.size());
                std::copy(vAddedNodes.cbegin(), vAddedNodes.cend(), std::back_inserter(lAddresses));
            }

    -   遍历所有的节点（`vNodes` 节点容器），进行下面处理。

        如果当前节点的地址是有效的，则加入 `mapConnected` map 中，Key 为当前节点的地址，值标明当前节点是否为入站节点。

        获取当前节点的地址名称。如果名称不空，则放进 `mapConnectedByName` map 中，Key 为当前节点的地址名称，值为一个 `std::pair` 对象，其中第一个值表明当前节点是否为入站节点，第二个值为节点的地址。

            std::map<CService, bool> mapConnected;
            std::map<std::string, std::pair<bool, CService>> mapConnectedByName;
            {
                LOCK(cs_vNodes);
                for (const CNode* pnode : vNodes) {
                    if (pnode->addr.IsValid()) {
                        mapConnected[pnode->addr] = pnode->fInbound;
                    }
                    std::string addrName = pnode->GetAddrName();
                    if (!addrName.empty()) {
                        mapConnectedByName[std::move(addrName)] = std::make_pair(pnode->fInbound, static_cast<const CService&>(pnode->addr));
                    }
                }
            }

    -   遍历 `lAddresses` 变量，进行下面处理。

        根据当前地址和当前网络类型，生成一个 `service` 对象，类型为 `CService`，和一个节点信息对象。

        如果当前地址是 IP:Port 形式，那么查找 `mapConnected` 集合对应的地址。如果可以找到，则设置节点信息对象的相关属性。

        如果当前地址是名称的形式，那么查找 `mapConnectedByName` 集合对应的地址。如果可以找到，则设置节点信息对象的相关属性。

        把当前地址信息对象加入 `ret` 集合中。

            for (const std::string& strAddNode : lAddresses) {
                CService service(LookupNumeric(strAddNode.c_str(), Params().GetDefaultPort()));
                AddedNodeInfo addedNode{strAddNode, CService(), false, false};
                if (service.IsValid()) {
                    // strAddNode is an IP:port
                    auto it = mapConnected.find(service);
                    if (it != mapConnected.end()) {
                        addedNode.resolvedAddress = service;
                        addedNode.fConnected = true;
                        addedNode.fInbound = it->second;
                    }
                } else {
                    // strAddNode is a name
                    auto it = mapConnectedByName.find(strAddNode);
                    if (it != mapConnectedByName.end()) {
                        addedNode.resolvedAddress = it->second.second;
                        addedNode.fConnected = true;
                        addedNode.fInbound = it->second.first;
                    }
                }
                ret.emplace_back(std::move(addedNode));
            }

        -   返回`ret` 集合。

2.  遍历所有的节点信息，如果当前节点还没有连接，进行下面的处理：

    生成地址对象 `addr`，类型为 `CAddress`。

    调用 `OpenNetworkConnection` 方法，连接到当前的节点。

        for (const AddedNodeInfo& info : vInfo) {
            if (!info.fConnected) {
                if (!grant.TryAcquire()) {
                    // If we've used up our semaphore and need a new one, let's not wait here since while we are waiting
                    // the addednodeinfo state might change.
                    break;
                }
                tried = true;
                CAddress addr(CService(), NODE_NONE);
                OpenNetworkConnection(addr, false, &grant, info.strAddedNode.c_str(), false, false, true);
                if (!interruptNet.sleep_for(std::chrono::milliseconds(500)))
                    return;
            }
        }


下面我们具体看下 `OpenNetworkConnection` 函数的处理。

1.  如果 `interruptNet` 为真，则返回。如果网络没有激活（`fNetworkActive` 为假），则返回。

        if (interruptNet) {
            return;
        }
        if (!fNetworkActive) {
            return;
        }

2.  如果参数 `pszDest` 为空（当前节点信息的地址），进一步，如果要连接的节点是本地的，或是已连接的，或是禁止的，则返回。如果参数 `pszDest` 不为空，进一步，如果节点是已连接的，则返回。

        if (!pszDest) {
            if (IsLocal(addrConnect) ||
                FindNode(static_cast<CNetAddr>(addrConnect)) || IsBanned(addrConnect) ||
                FindNode(addrConnect.ToStringIPPort()))
                return;
        } else if (FindNode(std::string(pszDest)))
            return;

3.  **调用 `ConnectNode` 方法，连接到指定地址，并返回对等节点 `CNode` 对象**。如果连接失败，则返回。

4.  如果参数 `grantOutbound` 对象存在，则调用其 `MoveTo` 方法，进行处理。

5.  如果参数 `fOneShot` 为真，则设置对等节点的 `fOneShot` 属性为真。

6.  如果是临时探测节点（参数`fFeeler` 为真），则设置对等节点的 `fFeeler` 属性为真。

7.  如果是手动连接的，则设置对等节点的 `m_manual_connection` 属性为真。

8.  **调用网络事件处理器的 `InitializeNode` 方法，进行对等节点初始化。**

    具体代码在 `net_processing.cpp` 文件的第 611 行，如下所示：

        void PeerLogicValidation::InitializeNode(CNode *pnode) {
            CAddress addr = pnode->addr;
            std::string addrName = pnode->GetAddrName();
            NodeId nodeid = pnode->GetId();
            {
                LOCK(cs_main);
                mapNodeState.emplace_hint(mapNodeState.end(), std::piecewise_construct, std::forward_as_tuple(nodeid), std::forward_as_tuple(addr, std::move(addrName)));
            }
            if(!pnode->fInbound)
                PushNodeVersion(pnode, connman, GetTime());
        }

    代码最主要的动作是，检查节点是否为出站节点，即连接到别的对等节点，如果是则调用 `PushNodeVersion` 方法，发送版本信息。具体消息消息处理部分。

9. **把生成的对等节点保存到 `vNodes` 向量中**。


### 3.1、ConnectNode

这个方法负责连接到具体的对等节点。我们来看下具体的处理。

1.  如果参数 `pszDest` 为空指针，则处理如下：

    如果要连接的地址是本地地址，则直接返回空指针。调用 `FindNode` 方法，查看指定的节点是否存在。如果存在，即已经连接，则返回空指针。

        if (pszDest == nullptr) {
            if (IsLocal(addrConnect))
                return nullptr;
            // Look for an existing connection
            CNode* pnode = FindNode(static_cast<CService>(addrConnect));
            if (pnode)
            {
                LogPrintf("Failed to open new connection, already connected\n");
                return nullptr;
            }
        }

2.  如果参数 `pszDest` 不是空指针，那么调用 `Lookup` 方法，查找/生成地址字符串对应的地址对象。如果找到，则进行下面的处理：

    生成要连接的地址对象。如果地址地址对象是无效的，则返回空指针。调用 `FindNode` 方法，查找对应的地址对象。如果存在，即已经连接，则返回空指针。

    这个地方解析要连接的地址字符串生成要连接的地址对象。

        const int default_port = Params().GetDefaultPort();
        if (pszDest) {
            std::vector<CService> resolved;
            if (Lookup(pszDest, resolved,  default_port, fNameLookup && !HaveNameProxy(), 256) && !resolved.empty()) {
                addrConnect = CAddress(resolved[GetRand(resolved.size())], NODE_NONE);
                if (!addrConnect.IsValid()) {
                    LogPrint(BCLog::NET, "Resolver returned invalid address %s for %s\n", addrConnect.ToString(), pszDest);
                    return nullptr;
                }
                LOCK(cs_vNodes);
                CNode* pnode = FindNode(static_cast<CService>(addrConnect));
                if (pnode)
                {
                    pnode->MaybeSetAddrName(std::string(pszDest));
                    LogPrintf("Failed to open new connection, already connected\n");
                    return nullptr;
                }
            }
        }

3.  如果要连接的地址对象是有效的，进行下面的处理。

    调用 `GetProxy` 方法，返回代理类型。如果方法返回为真，即存在代理，那么调用 `CreateSocket` 方法，创建代理套接字。如果成功创建，调用 `ConnectThroughProxy` 方法，通过代理连接到对等节点。

    如果不存在代理，那么调用 `CreateSocket` 方法，创建对等节点的套接字。如果成功创建，调用 `ConnectSocketDirectly` 方法，直接连接到对等节点。

        bool proxyConnectionFailed = false;

        if (GetProxy(addrConnect.GetNetwork(), proxy)) {
            hSocket = CreateSocket(proxy.proxy);
            if (hSocket == INVALID_SOCKET) {
                return nullptr;
            }
            connected = ConnectThroughProxy(proxy, addrConnect.ToStringIP(), addrConnect.GetPort(), hSocket, nConnectTimeout, &proxyConnectionFailed);
        } else {
            // no proxy needed (none set for target network)
            hSocket = CreateSocket(addrConnect);
            if (hSocket == INVALID_SOCKET) {
                return nullptr;
            }
            connected = ConnectSocketDirectly(addrConnect, hSocket, nConnectTimeout, manual_connection);
        }
        if (!proxyConnectionFailed) {
            // If a connection to the node was attempted, and failure (if any) is not caused by a problem connecting to
            // the proxy, mark this as an attempt.
            addrman.Attempt(addrConnect, fCountFailure);
        }

4.  如果要连接的字符串不空，且存在代理，那么：

    调用 `CreateSocket` 方法，生成代理的套接字。然后，调用 `ConnectThroughProxy` 方法，通过代理连接到指定的对等节点。

        hSocket = CreateSocket(proxy.proxy);
        if (hSocket == INVALID_SOCKET) {
            return nullptr;
        }
        std::string host;
        int port = default_port;
        SplitHostPort(std::string(pszDest), port, host);
        connected = ConnectThroughProxy(proxy, host, port, hSocket, nConnectTimeout, nullptr);

5.  如果以上都没有连接到主节点，则关闭套接字并返回空指针。

        if (!connected) {
            CloseSocket(hSocket);
            return nullptr;
        }

6.  最后，生成并返回主节点对象。

        NodeId id = GetNewNodeId();
        uint64_t nonce = GetDeterministicRandomizer(RANDOMIZER_ID_LOCALHOSTNONCE).Write(id).Finalize();
        CAddress addr_bind = GetBindAddress(hSocket);
        CNode* pnode = new CNode(id, nLocalServices, GetBestHeight(), hSocket, addrConnect, CalculateKeyedNetGroup(addrConnect), nonce, addr_bind, pszDest ? pszDest : "", false);
        pnode->AddRef();
        return pnode;
