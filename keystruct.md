#   比特币关键数据结构


##  1、共识参数

##  2、节点状态

节点状态由 `CNodeState` 结构体保存，分别由以下信息：

-   fCurrentlyConnected

    是否已经完全建立连接。

-   nMisbehavior

    不良节点的积分

-   fShouldBan

    节点是否应该被禁止，除非是白名单

-   rejects

-   pindexBestKnownBlock

-   hashLastUnknownBlock

    对等节点发布的最后一个未知区块的哈希值。

-   pindexLastCommonBlock

-   pindexBestHeaderSent

-   nUnconnectingHeaders

-   fSyncStarted

-   nStallingSince 

    停止下载区块的时间。
    
-   vBlocksInFlight

-   nDownloadingSince

-   fPreferHeaders

    布尔值，对于区块公告，对等接点愿意接收 invs 还是 headers 消息。

-   fPreferHeaderAndIDs

    布尔值，对于区块公告，对等接点愿意接收 invs 还是 cmpctblocks 消息。


##  3、对等节点信息

对等节点信息用 `CNode` 类来表示，重要的字段有下面一些。

-   vBlockHashesToAnnounce

    使用区块头部进行区块公告的集合。

