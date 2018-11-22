#   引子

创建钱包及其重要的几个都写完了，但总感觉还缺少一些什么，到底缺少什么呢？在重新读过几边之后，才发现原来是不能把一切连贯起来，所以又补写了本篇，作为创建钱包的补充说明。

#   正文

首先说明，下文中说的 `hash160 字符串` 指的是私钥或公钥的内容通过双重哈希算法（行 `SHA256`，再 `RIPEMD160`）生成的长度为 20 个字节的字符串。

钱包是 keystore 的扩展，管理着交易和余额，提供创建交易的能力。它有几个特别重要的属性，现在解释如下：

-   hdChain，HD 数据模型，它包含了一个 hash160 的种子，一个内部链的数量和一个外部链的数量。

-   setInternalKeyPool，内部密钥池的集合。

-   setExternalKeyPool，外部密钥池的集合。

-   set_pre_split_keypool，一个预分割的密钥池集合。

-   m_max_keypool_index，最大密钥池的索引。

-   m_pool_key_to_index

-   mapKeyMetadata，公钥元数据的映射集合。键为一个 hash160 字符串，值为一个公钥元数据。

-   m_script_metadata

-   mapMasterKeys，

-   mapWallet

-   mapAddressBook


钱包说完了，我们来看 keystore。keystore 顾名思议它代表了密钥的存储，自然而然提供了一些管理密钥的方法，比如：添加一个密钥（私钥和公钥）到 store中、检查给定地址对应的密钥是否在 store中、添加及检查一般脚本与只读脚本的功能等。根据是否加密，keystore 分为基础的和加密的两种。钱包继承自加密的 keystore，而加密的 keystore 又继承了基础的 keystore。

我们先来看下 基础 keystore ，它有以下几个重要的属性：

-   mapKeys，一个私钥映射集合。键为一个 hash160 字符串，值为一个私钥。

-   mapWatchKeys，一个只读的公钥映射集合。键为一个 hash160 字符串，值为一个公钥。

-   mapScripts，一个脚本映射集合。键为一个 hash160 字符串，值为一个用在交易输入和输出的序列化的脚本。

-   setWatchOnly，一个用在交易输入和输出的序列化的脚本集合。

加密 keystore 在基础 keystore 上增加了几个与加密相关的属性，它们分别为：

-   fUseCrypto，一个标志钱包是否为加密的变量。

-   vMasterKey，一个在加密情况下使用的私钥集合。当加密时 `mapKeys` 集合就会为空，而 `vMasterKey` 集合不空；当不加密时，情况正好反过来，即 `mapKeys` 集合不空，而 `vMasterKey` 集合为空。

-   mapCryptedKeys，一个映射集合。键为一个 hash160 字符串，值为一个公钥和加密后私钥组成的 pair 。


说完钱包与 keystore，下面我们就来看下密钥池。其实密钥池这个名字不准确，因为它仅包含了一个公钥，初次之外，还包含了两个布尔变量：`fInternal`、`m_pre_split`，前者表示密钥池是内部还是外部的，后者功能暂时不清楚，英语备注为：For keys generated before keypool split upgrade。`m_pre_split` 属性默认为假。



当创建钱包时，会执行如下几个动作：

1.  首先，生成一个私钥，根据私钥通过椭圆曲线算法生成对应的公钥；

2.  其次，也会生成对应的密钥元数据，并且把公钥的内容用 hash160 （先 SHA256，再RIPEMD160）算法生成的 20个字节的字符串保存为密钥元数据的种子，然后把私钥元数据保存在 `mapKeyMetadata` 集合中；

3.  然后，私钥被保存在 `mapKeys`，或 `mapCryptedKeys` 与 `vMasterKey` 集合中；当公钥是压缩的（通常是）时，通过公钥生成脚本，脚本进而被保存在 `mapScripts` 集合中；同时，私钥、公钥及密钥元数据都被保存在数据库中。

4.  再然后，公钥被作为种子，生成 HD 链对象，保存在钱包中。

5.  最后，当前面一切完成后，使用前面 3步生成的公钥做为种子，开始衍生用户指定数量的子私钥/公钥对，如果用户没有指定则默认衍生 3000 个子私钥/公钥对。

    -   衍生的私钥、公钥及元数据的处理与第 2、3 步相同；

    -   同时，用公钥生成的密钥池也被保存在数据库中；

    -   根据生成的私钥属于内部或外部，对应的索引保存在 `setInternalKeyPool`、或 `setExternalKeyPool` 集合中；不区分地，索引被保存在 `m_pool_key_to_index` 映射集合中，其中键为公钥对应的 hash160 字符串。

    > 其中，2000 个子私钥的路径从 `m/0'/0'/0` 到 `m/0'/0'/1999`，1000 个子私钥的路径从 `m/0'/1'/0` 到 `m/0'/1'/999`。其中的 `m` 代表私钥，`m/0'` 代表主私钥的第 1 个强化子私钥，`m/0'/0'/0` 代表主私钥的第 1 个强化子私钥的外部链的第 1 个强化孙私钥，同理，`m/0'/0'/1999` 代表主私钥的第 1 个强化子私钥的外部链的第 1999 个强化孙私钥；`m/0'/1'/0` 代表主私钥的第 1 个强化子私钥的内部链的第 1 个强化孙私钥，同理，`m/0'/1'/999` 代表主私钥的第 1 个强化子私钥的内部链的第 999 个强化孙私钥。


在上面过程中，1-3 步是 `GenerateNewSeed` 方法的主要内容，第 4 步是 `SetHDSeed` 的主要内容，第 5 步是 `TopUpKeyPool` 的主要内容。

最后，用两张图来概述 HD 钱包的创建。

![创建钱包](http://blockchain.szdyfjh.com/generate-hd-wallet.jpg)

![创建钱包](http://blockchain.szdyfjh.com/generate-hd-wallet.png)
