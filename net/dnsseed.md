#   P2P 网络建立之 DNS 种子节点处理

P2P 网络的建立是在系统启动的第 12 步，最后时刻调用 `CConnman::Start` 方法开始的。

本部分内容在 `net.cpp`、`net_processing.cpp` 等文件中。


## ThreadDNSAddressSeed

这个线程的目标是，只有在需要地址时才查询 DNS 种子，当我们不需要 DNS 种子时，会避免 DNS 种子查询。这样可以通过创建更少的识别 DNS 请求来提高用户隐私，通过减少种子对网络拓扑的影响来减少信任，并减少种子的流量。

线程定义在 `net.cpp` 文件的 1603 行。下面我们开始进行具体的解读。

1.  如果对等节点的数量大于 0，且没有指定 `-forcednsseed`，或指定了但值为 `false`，进行下面的处理：

    遍历所有的节点，如果节点已成功连接，且不是引导节点，且 `fOneShot` 属性为假，且不是不是手动连接的，且不是入站节点，那么变量 `nRelevant` 加1。

    如果变量 `nRelevant` 大于2，即 P2P 网络已经可用，则退出函数。

        if ((addrman.size() > 0) &&
            (!gArgs.GetBoolArg("-forcednsseed", DEFAULT_FORCEDNSSEED))) {
            if (!interruptNet.sleep_for(std::chrono::seconds(11)))
                return;

            LOCK(cs_vNodes);
            int nRelevant = 0;
            for (auto pnode : vNodes) {
                nRelevant += pnode->fSuccessfullyConnected && !pnode->fFeeler && !pnode->fOneShot && !pnode->m_manual_connection && !pnode->fInbound;
            }
            if (nRelevant >= 2) {
                LogPrintf("P2P peers available. Skipped DNS seeding.\n");
                return;
            }
        }

2.  获取并遍历所有的 DNS 种子节点。

        for (const std::string &seed : vSeeds) {
            if (interruptNet) {
                return;
            }
            if (HaveNameProxy()) {
                AddOneShot(seed);
            } else {
                std::vector<CNetAddr> vIPs;
                std::vector<CAddress> vAdd;
                ServiceFlags requiredServiceBits = GetDesirableServiceFlags(NODE_NONE);
                std::string host = strprintf("x%x.%s", requiredServiceBits, seed);
                CNetAddr resolveSource;
                if (!resolveSource.SetInternal(host)) {
                    continue;
                }
                unsigned int nMaxIPs = 256; // Limits number of IPs learned from a DNS seed
                if (LookupHost(host.c_str(), vIPs, nMaxIPs, true))
                {
                    for (const CNetAddr& ip : vIPs)
                    {
                        int nOneDay = 24*3600;
                        CAddress addr = CAddress(CService(ip, Params().GetDefaultPort()), requiredServiceBits);
                        addr.nTime = GetTime() - 3*nOneDay - GetRand(4*nOneDay); // use a random age between 3 and 7 days old
                        vAdd.push_back(addr);
                        found++;
                    }
                    addrman.Add(vAdd, resolveSource);
                } else {
                    // We now avoid directly using results from DNS Seeds which do not support service bit filtering,
                    // instead using them as a oneshot to get nodes with our desired service bits.
                    AddOneShot(seed);
                }
            }
        }

    下面，对上面的代码进行讲解。

    如果指定了代理，则调用 `AddOneShot` 方法，保存当前 DNS 种子节点到 `vOneShots` 集合中。否则，进行下面的处理：

    -   生成两个集合 `vIPs`、`vAdd` 。`vIPs` 集合中保存的是 `CNetAddr` 对象，代表了一个IP地址。`vAdd`集合中保存的是 `CAddress` 对象，`CAddress` 继承自 `CService`，后者又继承自 `CAddress`，包含了一些关于对等节点别的信息。

    -   调用 `GetDesirableServiceFlags` 方法，获得服务标志位。

    -   调用 `strprintf` 函数，格式化 DNS 种子节点的地址。

        `strprintf` 是一个宏定义，实际调用的是 Boost 库的 `tfm::format`。

    -   生成类型为 `CNetAddr` 的地址对象 `resolveSource`，并调用其 `SetInternal` 方法，设置 `resolveSource` 的 IP。如果出错，则返回处理下一个。

    -   调用 `LookupHost` 方法，根据 DNS 种子节点获取其保存的对等节点列表。并保存在 `vIPs` 集合中。

        `LookupHost` 方法内部主要调用了 `LookupIntern` 方法进行处理。下面我们看下后者的具体处理。

        -   生成一个地址对象 `addr`。然后调用其 `SetSpecial` 方法进行处理。

            在该方法内部，如果 DNS种子节点不是以 `.onion` 结尾，即不是暗网地址，则直接返回假。否则进行下面的处理。

            调用 `DecodeBase32` 方法，解析不包括暗网后缀在内的具体的地址。接下来，检查地址的长度是否不等于指定的长度，如果是则返回假。否则，对地址进行处理并转化为IP地址，然后返回真。

                bool CNetAddr::SetSpecial(const std::string &strName)
                {
                    if (strName.size()>6 && strName.substr(strName.size() - 6, 6) == ".onion") {
                        std::vector<unsigned char> vchAddr = DecodeBase32(strName.substr(0, strName.size() - 6).c_str());
                        if (vchAddr.size() != 16-sizeof(pchOnionCat))
                            return false;
                        memcpy(ip, pchOnionCat, sizeof(pchOnionCat));
                        for (unsigned int i=0; i<16-sizeof(pchOnionCat); i++)
                            ip[i + sizeof(pchOnionCat)] = vchAddr[i];
                        return true;
                    }
                    return false;
                }

            如果前面方法返回的结果为真，即 DNS 种子为暗网地址，则把当前地址加入 `vIP` 集合，并返回。

                CNetAddr addr;
                if (addr.SetSpecial(std::string(pszName))) {
                    vIP.push_back(addr);
                    return true;
                }

        -   生成一个类型为 `addrinfo` 的结构体对象 aiHint，并设置其各个属性值。

        -   生成一个类型为 `addrinfo` 的结构体对象 aiRes，然后调用 `getaddrinfo` 方法，根据 DNS 种子节点来获取一个地址链接表。

            这个方法是系统提供的方法，返回的是一个 sockaddr 结构的链表而不是一个地址清单。第一个参数是一个主机名或者地址串，第二个参数是一个服务名或者10进制端口号数串，第三个参数可以是一个空指针，也可以是一个指向某个addrinfo结构的指针，调用者在这个结构中填入关于期望返回的信息类型的暗示，最后一个参数是返回的结果。

                int nErr = getaddrinfo(pszName, nullptr, &aiHint, &aiRes);
                if (nErr)
                    return false;

        -   接下来只要地址信息链表不空，且当前获取的对等节点IP数量小于指定的数量或者指定的数量是0（即不限制对等节点的数量），就循环这个链表进行下面的处理。

            根据返回的地址信息对象，是IPV4 或者是 IPV6，生成生成不同的 `CNetAddr` 对象。如果这个地址对象不是内部 IP，则保存到 `vIP` 集合中。从地址信息链表中取得下一个地址信息对象。

                struct addrinfo *aiTrav = aiRes;
                while (aiTrav != nullptr && (nMaxSolutions == 0 || vIP.size() < nMaxSolutions))
                {
                    CNetAddr resolved;
                    if (aiTrav->ai_family == AF_INET)
                    {
                        assert(aiTrav->ai_addrlen >= sizeof(sockaddr_in));
                        resolved = CNetAddr(((struct sockaddr_in*)(aiTrav->ai_addr))->sin_addr);
                    }

                    if (aiTrav->ai_family == AF_INET6)
                    {
                        assert(aiTrav->ai_addrlen >= sizeof(sockaddr_in6));
                        struct sockaddr_in6* s6 = (struct sockaddr_in6*) aiTrav->ai_addr;
                        resolved = CNetAddr(s6->sin6_addr, s6->sin6_scope_id);
                    }
                    /* Never allow resolving to an internal address. Consider any such result invalid */
                    if (!resolved.IsInternal()) {
                        vIP.push_back(resolved);
                    }

                    aiTrav = aiTrav->ai_next;
                }

        -   调用 `freeaddrinfo` 方法，释放 `getaddrinfo` 方法所申请的内存空间。

        -   根据 `vIP` 集合的大小，返回真假。

    -   如果 `LookupHost` 方法返回结果为真，即根据当前 DNS 种子节点查找到了至少一个对等节点，则进行下面的处理。

        遍历 `vIPs` 集合，根据当前的 IP 地址，生成一个 `CAddress` 地址对象，并保存在 `vAdd` 集合中，同时把代表找到节点的变量 `found` 加1。

        调用地址管理器的 `Add` 方法，保存多个地址。

        具体代码如下：

            for (const CNetAddr& ip : vIPs)
            {
                int nOneDay = 24*3600;
                CAddress addr = CAddress(CService(ip, Params().GetDefaultPort()), requiredServiceBits);
                addr.nTime = GetTime() - 3*nOneDay - GetRand(4*nOneDay); // use a random age between 3 and 7 days old
                vAdd.push_back(addr);
                found++;
            }
            addrman.Add(vAdd, resolveSource);

    -   如果 `LookupHost` 方法返回结果为假，即根据当前 DNS 种子节点没找到一个对等节点，则调用 `AddOneShot` 方法进行处理。

        `AddOneShot` 方法内部简单地把当前 DNS 种子加入 `vOneShots` 集合。


###  2.1、CAddrMan::Add 方法

下面我们对地址管理器的 `Add` 方法做下介绍。这个方法位于 `addrman.h` 文件的 540 行。

这个方法主体是一个 for 循环，遍历 `CAddress` 集合，针对每一个 `CAddress` 对象调用 `Add_` 方法进行处理。并返回是否添加成功。代码如下：

    bool Add(const std::vector<CAddress> &vAddr, const CNetAddr& source, int64_t nTimePenalty = 0)
    {
        LOCK(cs);
        int nAdd = 0;
        Check();
        for (std::vector<CAddress>::const_iterator it = vAddr.begin(); it != vAddr.end(); it++)
            nAdd += Add_(*it, source, nTimePenalty) ? 1 : 0;
        Check();
        if (nAdd) {
            LogPrint(BCLog::ADDRMAN, "Added %i addresses from %s: %i tried, %i new\n", nAdd, source.ToString(), nTried, nNew);
        }
        return nAdd > 0;
    }

接下来，我们来看一下 `Add_` 方法。这个方法在 `addrman.cpp` 文件的第254行。

1.  如果当前地址是不可路由的，则直接返回假。

        if (!addr.IsRoutable())
            return false;

2.  调用 `Find` 方法，根据地址对象找到其对应的地址信息。

        std::map<CNetAddr, int>::iterator it = mapAddr.find(addr);
        if (it == mapAddr.end())
            return nullptr;
        if (pnId)
            *pnId = (*it).second;
        std::map<int, CAddrInfo>::iterator it2 = mapInfo.find((*it).second);
        if (it2 != mapInfo.end())
            return &(*it2).second;
        return nullptr;

3.  如果地址对象来源对象，设置变量 `nTimePenalty` 等于0。

4.  如果找到对应的地址信息，则设置地址信息的相关属性

        bool fCurrentlyOnline = (GetAdjustedTime() - addr.nTime < 24 * 60 * 60);
        int64_t nUpdateInterval = (fCurrentlyOnline ? 60 * 60 : 24 * 60 * 60);
        if (addr.nTime && (!pinfo->nTime || pinfo->nTime < addr.nTime - nUpdateInterval - nTimePenalty))
            pinfo->nTime = std::max((int64_t)0, addr.nTime - nTimePenalty);

        // add services
        pinfo->nServices = ServiceFlags(pinfo->nServices | addr.nServices);

        // do not update if no new information is present
        if (!addr.nTime || (pinfo->nTime && addr.nTime <= pinfo->nTime))
            return false;

        // do not update if the entry was already in the "tried" table
        if (pinfo->fInTried)
            return false;

        // do not update if the max reference count is reached
        if (pinfo->nRefCount == ADDRMAN_NEW_BUCKETS_PER_ADDRESS)
            return false;

        // stochastic test: previous nRefCount == N: 2^N times harder to increase it
        int nFactor = 1;
        for (int n = 0; n < pinfo->nRefCount; n++)
            nFactor *= 2;
        if (nFactor > 1 && (RandomInt(nFactor) != 0))
            return false;

5.  如果没有找到对应的地址信息，则生成新的地址信息。

        pinfo = Create(addr, source, &nId);
        pinfo->nTime = std::max((int64_t)0, (int64_t)pinfo->nTime - nTimePenalty);
        nNew++;
        fNew = true;

    在 `Create` 方法中，生成一个新的 `CAddrInfo` 对象，并放到 `mapInfo` 集合中，同时在在 `mapAddr` 集合中增加对应的条目。具体代码如下：

        int nId = nIdCount++;
        mapInfo[nId] = CAddrInfo(addr, addrSource);
        mapAddr[addr] = nId;
        mapInfo[nId].nRandomPos = vRandom.size();
        vRandom.push_back(nId);
        if (pnId)
            *pnId = nId;
        return &mapInfo[nId];

6.  接下来处理其他一些信息，代码比较简单不详述。

        int nUBucket = pinfo->GetNewBucket(nKey, source);
        int nUBucketPos = pinfo->GetBucketPosition(nKey, true, nUBucket);
        if (vvNew[nUBucket][nUBucketPos] != nId) {
            bool fInsert = vvNew[nUBucket][nUBucketPos] == -1;
            if (!fInsert) {
                CAddrInfo& infoExisting = mapInfo[vvNew[nUBucket][nUBucketPos]];
                if (infoExisting.IsTerrible() || (infoExisting.nRefCount > 1 && pinfo->nRefCount == 0)) {
                    // Overwrite the existing new table entry.
                    fInsert = true;
                }
            }
            if (fInsert) {
                ClearNew(nUBucket, nUBucketPos);
                pinfo->nRefCount++;
                vvNew[nUBucket][nUBucketPos] = nId;
            } else {
                if (pinfo->nRefCount == 0) {
                    Delete(nId);
                }
            }
        }

7.  返回真。
