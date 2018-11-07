#   比特币关键数据结构节点状态

节点状态由 `CNodeState` 结构体保存，分别由以下信息：

-   fCurrentlyConnected

    是否已经完全建立连接。接收 `verack` 消息时，设置的。

-   nMisbehavior

    节点的不良积分

-   fShouldBan

    节点是否应该被禁止，除非是白名单

-   rejects

-   pindexBestKnownBlock

-   hashLastUnknownBlock

    对等节点发布的最后一个未知区块的哈希值。

-   pindexLastCommonBlock

-   pindexBestHeaderSent

    发送给对等节点的最佳区块索引。

-   nUnconnectingHeaders

    未连接到区块链上的头部数量。
    
-   fSyncStarted

-   nStallingSince 

    停止下载区块的时间。
    
-   vBlocksInFlight

-   nDownloadingSince

-   fPreferHeaders

    布尔值，对于区块公告，对等接点愿意接收 invs 还是 headers 消息。

-   fPreferHeaderAndIDs

    布尔值，对于区块公告，对等接点愿意接收 invs 还是 cmpctblocks 消息。

-	fHaveWitness

	布尔型，节点是否支持隔离见证。接收版本消息时设置的。

-	fPreferredDownload

	节点是否为首选的下载节点。

-	fProvidesHeaderAndIDs

	布尔型，当我们请求时，节点是否可以返回 cmpctblocks 给我们。
