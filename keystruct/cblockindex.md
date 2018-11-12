#   比特币关键数据结构之区块索引对象

区块索引对象用 `CBlockIndex` 类来表示，重要字段有下面一些。

-	phashBlock

	指向区块的哈希

-	pprev

	指向前一个区块的索引指针，类型为 ` CBlockIndex*`。

-	pskip

	指向更前一个区块的索引指针，类型为 ` CBlockIndex*`。

-	nHeight

	区块在区块链中的高度，创世区块调试为0。

-	nFile

	区块存在哪一个 `blk?????.dat` 文件中。

-	nDataPos

	区块在 `blk?????.dat` 文件中的偏移量。

-	nUndoPos

	区块的 undo data 在 `rev?????.dat` 文件中的偏移量。

-	nChainWork

	包含当前区块在内的总的工作量，只存在于内存中。

-	nTx

	区块的交易数量，只存在于内存中。

-	nChainTx

	包含当前区块在内的总的交易数量，只存在于内存中。

-	nStatus

	区块的验证状态。可能的值为：

	-	BLOCK_VALID_HEADER

		版本 OK，哈希满足声称的 PoW，交易数大于等于1 小于等于最大数量，时间戳不在未来。

	-	BLOCK_VALID_TREE

		所有的父区块头部已经找到，难度也匹配，时间戳大于等于 median previous。隐含着所有父亲都在树上。

	-	BLOCK_VALID_TRANSACTIONS

		只有第一个交易是 coinbase，coinbase 输入脚本的长度大于等于 2 小于等于 100，所有交易是 OK 的，没有重复的交易 ID，sigops, size, merkle root。

	-	BLOCK_VALID_CHAIN

		输出没有超过输入，没有双重花费，coinbase 输出是 OK 的，没有使用时间过短的 coinbase，BIP30。

	-	BLOCK_VALID_SCRIPTS

		脚本和签名是 Ok 的。

	-	BLOCK_VALID_MASK

		所有有效位，等于 `BLOCK_VALID_HEADER | BLOCK_VALID_TREE | BLOCK_VALID_TRANSACTIONS | BLOCK_VALID_CHAIN | BLOCK_VALID_SCRIPTS`

	-	BLOCK_HAVE_DATA

		所有区块在 `blk*.dat` 文件中可用。

	-	BLOCK_HAVE_UNDO

		undo data available in rev*.dat

	-	BLOCK_HAVE_MASK

		等于 `BLOCK_HAVE_DATA | BLOCK_HAVE_UNDO`

	-	BLOCK_FAILED_VALID

		最后达到有效性后的阶段失败

	-	BLOCK_FAILED_CHILD

		descends from failed block

	-	BLOCK_FAILED_MASK

		等于 `BLOCK_FAILED_VALID | BLOCK_FAILED_CHILD`

	-	BLOCK_OPT_WITNESS

		block data in blk*.data was received with a witness-enforcing client

-	nVersion

	区块头部的版本号。

-	hashMerkleRoot

	区块头部的默克尔根。

-	nTime

	区块头部的时间戳。

-	nBits

	区块头部的难度数。

-	nNonce

	区块头部的随机数。

-	nSequenceId

-	nTimeMax


