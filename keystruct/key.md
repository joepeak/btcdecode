#   密钥相关

##  1、私钥

系统中使用 `CKey` 来表示私钥的封装。。普通私钥长度为 279，压缩私钥长度为 214。

具有下列重要属性：

-   fValid

    私钥是否有效

-   fCompressed

    是否（将）压缩对应于该私钥的公钥

-   keydata

    真实的数据


##  2、CExtKey

扩展私钥，具有下列属性：

1.  nDepth

2.  vchFingerprint

3.  nChild

4.  chaincode

5.  key


##  3、公钥

系统中使用类 `CPubKey` 来表示公钥。普通公钥长度为 65，压缩公钥长度为 33。

签名长度为 72，压缩签名长度为 65。

公钥内部使用一个无符号字符数组 `vch` ，来存储序列化的数据。它的第一个字符是一个标志，如果等于 2 ，或3 那么公钥的长度就是压缩的长度，如果等于 4，或6，或7 那么公钥长度就是无压缩的长度，其他长度都是 0。

`GetLen`、`size` 都是根据 `vch` 数组的第一个字符来判断的，`Invalidate` 设置公钥无效时，设置数组的第一个字符为 `0xFF`

下面对某些方法做简单说明：

1.  GetID 方法

    生成并返回一个 CKeyID 对象。先调用 `Hash160` 方法，计算参数 160 位的哈希，内部分别使用 `SHA256` 和 `RIPEMD160` 进行双重哈希。

2.  GetHash

    返回公钥 256 位的哈希，内部调用 `SHA256` 来生成哈希的。



##  4、CExtPubKey

扩展公钥，具有下列属性：

-   nDepth

-   vchFingerprint

-   nChild

-   chaincode

-   pubkey


##  5、CKeyID

一个私钥 `CKey` 的引用。


##  6、密钥池

密钥池由类 `CKeyPool` 表示，具有下列重要属性：

-   nTime

-   vchPubKey

    一个公钥对象。生成地址时候，设置的。

-   fInternal

-   m_pre_split

