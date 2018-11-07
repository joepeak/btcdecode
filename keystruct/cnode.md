#   比特币关键数据结构之对等节点信息

对等节点信息用 `CNode` 类来表示，重要的字段有下面一些。

-	nServices

	节点支持的服务，接收 `version` 消息时，设置的。当前支持的服务有：

	-	NODE_NETWORK

		节点可以提供完整的区块而不仅仅是区块头。所有没有进行修剪的比特币核心客户端都是这个值，SPV客户端和其他轻量级客户端不设置这个值。

	-	NODE_GETUTXO

		节点支持 `getutxo` 协议请求，比特币核心客户端不支持这个，但  Bitcoin XT 补丁支持这个。详见 BIP 0064。

	-	NODE_BLOOM

		节点有能力并且愿意支持布隆过滤器，比特币核心默认支持此功能而不用公告这个位。详见 BIP 0111。

	-	NODE_WITNESS

		节点可以响应包含隔离见证的区块和交易请求。详见 BIP 0144。

	-	NODE_XTHIN

		节点支持 Xtreme Thinblocks。

	-	NODE_NETWORK_LIMITED

		基本等于 `NODE_NETWORK`，除了只提供 288 个区块，即2天内的区块。

-	hSocket

-	nSendSize

-	nSendOffset

-	nSendBytes

-	vSendMsg

-	vProcessMsg

-	nProcessQueueSize

-	vRecvGetData

	接收到的请求数据队列，接收 `getdata` 消息时，设置的。
-	nRecvBytes

-	nRecvVersion

	版本信息，服务器发送的版本与 70015 这个版本的小者，接收 `verack` 消息时，设置的。

-	nLastSend

-	nLastRecv

-	nTimeConnected

-	nTimeOffset

-	addr

	对待节点的地址对象，类型为 `CAddress`。

-	addrBind

-	nVersion

	客户端发送的版本，接收 `version` 消息时，设置的。

-	fWhitelisted

	布尔型，节点是否为白名单节点

-	fFeeler

	布尔型，节点是否为临时的引导节点

-	fOneShot

-	m_manual_connection

	布尔型，节点是否为手工指定连接的

-	fClient

	布尔型，节点是否可以提供区块服务。如果节点没有设置 `NODE_NETWORK`、或 `NODE_NETWORK_LIMITED` ，这个值为假，即不能提供区块服务。如果设置其中一个，这个值就为真，即可以提供区块服务。

-	m_limited_node

	布尔型，节点是否为限制型节点。节点设置了 `NODE_NETWORK_LIMITED`，但没有设置 `NODE_NETWORK`，这值就为真。

-	fInbound

	布尔型，节点是否为入站节点

-	fSuccessfullyConnected

	布尔型，对等节点完全连接成功。只有在收到版本确认时，才会设置这个标志。

-	fDisconnect

	布尔型，节点是否已经断开

-	fRelayTxes

	布尔型，是否中继交易，接收 `version` 消息时，设置的。

-	fSentAddr

	是否发送过请求地址标志。

-	grantOutbound

-	pfilter

	类型为 CBloomFilter 的智能指针，在接收 `filterload` 消息时设置。
	
-	nRefCount

-	nKeyedNetGroup

-	fPauseRecv

-	fPauseSend

-	mapSendBytesPerMsgCmd

-	mapRecvBytesPerMsgCmd

-	hashContinue

-	nStartingHeight

	节点最后一个区块的高度，也即节点开始接收区块的高度，接收 `version` 消息时，设置的。

-	vAddrToSend

	等待发送的地址，自身要发送的地址也放在这个集合中。

-	addrKnown

	已知的地址集合，类型为 `CRollingBloomFilter` 。

-	fGetAddr

	布尔型，是否向这个节点请求过地址。

-	setKnown

-	nNextAddrSend

-	nNextLocalAddrSend

-	filterInventoryKnown

-	setInventoryTxToSend

-	vInventoryBlockToSend

-	setAskFor

-	mapAskFor

-	nNextInvSend

-	vBlockHashesToAnnounce

	使用区块头部进行区块公告的集合

-	fSendMempool

-	timeLastMempoolReq

-	nLastBlockTime

-	nLastTXTime

-	nPingNonceSent

	无符号 64 位，期待的 pong 中继，如果是 0，就是非期待的。

-	nPingUsecStart

-	nPingUsecTime

-	nMinPingUsecTime

-	fPingQueued

-	minFeeFilter

-	lastSentFeeFilter

-	nextSendTimeFeeFilter

-	nLocalHostNonce

	在进行 `version` 时，生成的随机值。

-	nLocalServices

-	nMyStartingHeight

-	nSendVersion

	版本信息，客户端发送的版本与 70015 这个版本的小者，接收 `version` 消息时，设置的。

-	vRecvMsg

-	addrName

-	addrLocal

	对等节点的地址，接收 `version` 消息时，设置的。

-	hashContinue

-	nLastTXTime

	最后一次收到交易的时间

-	nLastBlockTime

	最后一次收到区块的时间

