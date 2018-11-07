#   比特币关键数据结构之区块相关

##	区块对象

区块对象用 `CBlock` 类来表示，继承自 `CBlockHeader` ，重要字段如下：

-	vtx

	交易对象的向量集。

-	fChecked

	是否已经检查。只保存在内存中。

##	区块头部

区块头部对象用 `CBlockHeader` 类表示，重要字段如下：

-	nVersion

	区块的版本号。

-	hashPrevBlock

	前一个区块的哈希。

-	hashMerkleRoot

	区块的默克尔数。

-	nTime

	区块生成时间。

-	nBits

	区块的难度数。

-	nNonce

	区块的随机数。

