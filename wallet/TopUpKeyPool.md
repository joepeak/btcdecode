#   写在前面

创建钱包和交易是比特币最重要的两方面，涉及到很多很多的内容，远非一篇文章能概括的完。上一篇从整体上讲解了钱包的创建流程，虽然已经很详细了，但还是漏掉了几个重要的方法，从本篇开始我们讲解这几个特别重要的方法。对钱包感兴趣的朋友一定不要错误。


#   TopUpKeyPool 填充密钥池

本方法在第一次创建时会执行，在升级钱包到 HD 时，也会执行。它被用来填充密钥池 keypool。下面我们来看下方法的执行。

1.  如果标志禁止私钥，则返回假。

        if (IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS)) {
            return false;
        }

2.  如果钱包被锁定，则返回假。

        if (IsLocked())
            return false;

3.  如果参数 `kpSize` 大于0，则设置变量 `nTargetSize` 为 `kpSize`；否则，设置为用户指定的值，或默认的 1000。

        unsigned int nTargetSize;
        if (kpSize > 0)
            nTargetSize = kpSize;
        else
            nTargetSize = std::max(gArgs.GetArg("-keypool", DEFAULT_KEYPOOL_SIZE), (int64_t) 0);

4.  计算内部、外部可用的密钥数量。

        int64_t missingExternal = std::max(std::max((int64_t) nTargetSize, (int64_t) 1) - (int64_t)setExternalKeyPool.size(), (int64_t) 0);
        int64_t missingInternal = std::max(std::max((int64_t) nTargetSize, (int64_t) 1) - (int64_t)setInternalKeyPool.size(), (int64_t) 0);

    在用户不指定的情况下，并且是第一次，因为 `setExternalKeyPool`、`setInternalKeyPool` 这两个集合都为空，所以 `missingExternal`、`missingInternal` 两个变量的值都为 1000。

5.  如果不支持 HD 钱包，或者不支持 HD 分割，那么设置变量 `missingInternal` 为0。

        if (!IsHDEnabled() || !CanSupportFeature(FEATURE_HD_SPLIT))
        {
            missingInternal = 0;
        }

6.  生成访问数据库的对象。

        bool internal = false;
        WalletBatch batch(*database);

7.  进行 `for (int64_t i = missingInternal + missingExternal; i--;)` 循环。

    -   如果 `i` 小于 `missingInternal` ，则设置变量 `internal` 为真。

            if (i < missingInternal) {
                internal = true;
            }

    -   设置当前索引。

            int64_t index = ++m_max_keypool_index;

    -   生成公钥对象。

            CPubKey pubkey(GenerateNewKey(batch, internal));

        `GenerateNewKey` 方法用来生成公钥。我们来看下这个方法的执行流程。

        -   生成表示创建的公钥是否为压缩的变量 `fCompressed`。在 0.6 版本之后的公钥默认都是压缩的。

                bool fCompressed = CanSupportFeature(FEATURE_COMPRPUBKEY);

        -   创建私钥对象和密钥元数据对象。这时候私钥和密钥元数据对象都没有经过设置，还不能成为真正的私钥和密钥元数据，只有经过下面两个方法中的某一个处理之后，才变成真正可用的对象。

                CKey secret;
                int64_t nCreationTime = GetTime();
                CKeyMetadata metadata(nCreationTime);

        -   如果钱包支持 HD，那么调用 `DeriveNewChildKey` 方法来衍生子私钥，否则，调用 `MakeNewKey` 方法来生成私钥。

                if (IsHDEnabled()) {
                    DeriveNewChildKey(batch, metadata, secret, (CanSupportFeature(FEATURE_HD_SPLIT) ? internal : false));
                } else {
                    secret.MakeNewKey(fCompressed);
                }

            `IsHDEnabled` 是通过私钥对象的 `hdChain.seed_id` 对象是否为空来判断的，因为在前面生成钱包私钥之后，就设置了这个方法，所以这个方法一定返回真。

            `MakeNewKey` 这个方法，我们在前面创建钱包的私钥时已经看到过，`DeriveNewChildKey` 这个方法我们在下面进行重点讲解，这里暂且略过。

        -   如果变量 `fCompressed` 为真，则调用 `SetMinVersion` 方法，设置最小版本为 `FEATURE_COMPRPUBKEY`，支持压缩公钥的版本。

                if (fCompressed) {
                    SetMinVersion(FEATURE_COMPRPUBKEY);
                }

        -   调用私钥的 `GetPubKey` 方法，返回对应的公钥。

            方法内部使用椭圆曲线算法求得公钥。具体不细讲，读者自行看代码。

                CPubKey CKey::GetPubKey() const {
                    assert(fValid);
                    secp256k1_pubkey pubkey;
                    size_t clen = CPubKey::PUBLIC_KEY_SIZE;
                    CPubKey result;
                    int ret = secp256k1_ec_pubkey_create(secp256k1_context_sign, &pubkey, begin());
                    assert(ret);
                    secp256k1_ec_pubkey_serialize(secp256k1_context_sign, (unsigned char*)result.begin(), &clen, &pubkey, fCompressed ? SECP256K1_EC_COMPRESSED : SECP256K1_EC_UNCOMPRESSED);
                    assert(result.size() == clen);
                    assert(result.IsValid());
                    return result;
                }

        -   把密钥元数据加入 `mapKeyMetadata` 集合中。

                mapKeyMetadata[pubkey.GetID()] = metadata;

        -   调用 `UpdateTimeFirstKey` 方法，更新 `nTimeFirstKey` 属性。

                UpdateTimeFirstKey(nCreationTime);
                
        -   调用 `AddKeyPubKeyWithDB` 方法，把私钥和公钥保存到数据库中。这个方法在前面已经讲过，这里再讲了。

                if (!AddKeyPubKeyWithDB(batch, secret, pubkey)) {
                    throw std::runtime_error(std::string(__func__) + ": AddKey failed");
                }

        -   返回公钥。

    -   用公钥生成一个密钥池实体 `CKeyPool` 对象，并调用访问钱包数据库对象的 `WritePool` 方法，以 `pool` 为键把密钥池实体对象写入数据库。

            if (!batch.WritePool(index, CKeyPool(pubkey, internal))) {
                throw std::runtime_error(std::string(__func__) + ": writing generated key failed");
            }

    -   如果变量 `internal` 为真，则把索引保存到 `setInternalKeyPool` 集合中，否则，保存到 `setExternalKeyPool` 集合中。

            if (internal) {
                setInternalKeyPool.insert(index);
            } else {
                setExternalKeyPool.insert(index);
            }

    -   把索引保存到 `m_pool_key_to_index` 集合中。

            m_pool_key_to_index[pubkey.GetID()] = index;

8.  返回真。


##  DeriveNewChildKey 衍生子密钥

这个方法用来衍生子密钥。方法的逻辑如下：

1.  生成相关的变量。

        CKey seed;                     //seed (256bit)
        CExtKey masterKey;             //hd master key
        CExtKey accountKey;            //key at m/0'
        CExtKey chainChildKey;         //key at m/0'/0' (external) or m/0'/1' (internal)
        CExtKey childKey;              //key at m/0'/0'/<n>'

2.  调用 `GetKey` 方法，根据HD 链对象中保存的公钥对象来得到对应的私钥对象。这个私钥是根私钥，也被称为种子私钥。如果获得不到，则抛出异常。

        if (!GetKey(hdChain.seed_id, seed))
            throw std::runtime_error(std::string(__func__) + ": seed not found");

    `GetKey` 方法执行流程如下：

    -   如果钱包没有加密，则调用 `CBasicKeyStore::GetKey` 方法，返回公钥对象的私钥。

            if (!IsCrypted()) {
                return CBasicKeyStore::GetKey(address, keyOut);
            }

        `CBasicKeyStore::GetKey` 方法直接从 `mapKeys` 集合中取出对应的私钥。

        `mapKeys` 集合是我们前面分析 `DeriveNewSeed` 这个方法的第 6 步 `AddKeyPubKey` 这个方法中把私钥/公钥及元数据保存到数据库过程中设置的。

    -   否则，直接从加密集合 `mapCryptedKeys` 中取得对应的私钥。

            CryptedKeyMap::const_iterator mi = mapCryptedKeys.find(address);
            if (mi != mapCryptedKeys.end())
            {
                const CPubKey &vchPubKey = (*mi).second.first;
                const std::vector<unsigned char> &vchCryptedSecret = (*mi).second.second;
                return DecryptKey(vMasterKey, vchCryptedSecret, vchPubKey, keyOut);
            }

    -   如果以上两种情况都不能返回，那么只能返回假了。

3.  调用主私钥（扩展私钥）的 `SetSeed` 方法，设置种子。此时的主私钥只是一个对象，而不能当作真正的私钥来使用，因为内部数据不存在。只有在本方法调用之后，主私钥才能真正用来衍生密钥。

    我们现在来看下 `SetSeed` 方法的逻辑：

    -   首先，生成需要的变量。

            static const unsigned char hashkey[] = {'B','i','t','c','o','i','n',' ','s','e','e','d'};
            std::vector<unsigned char, secure_allocator<unsigned char>> vout(64);

    -   其次，调用 `CHMAC_SHA512` 方法，使用 `HMAC_SHA512` 算法，根据种子私钥生成长度为 512 位的字符串。

            CHMAC_SHA512(hashkey, sizeof(hashkey)).Write(seed, nSeedLen).Finalize(vout.data());

    -   然后，调用私钥的 `Set` 方法，用 512 位的字符串的左边 256 位来初始化私钥的 `keydata` 属性，并且设置私钥的有效标志 `fValid` 属性为真，压缩标志 `fCompressed` 为参数指定的值，默认为真。

            key.Set(vout.data(), vout.data() + 32, true);

    -   调用 `memcpy` 方法，把 512 位的字符串的右边 256 位保存为链码。

            memcpy(chaincode.begin(), vout.data() + 32, 32);

    -   设置主私钥的 `nDepth`、`nChild` 为 0。

            nDepth = 0;
            nChild = 0;

    -   把 `vchFingerprint` 重置为 0。

            memset(vchFingerprint, 0, sizeof(vchFingerprint));

    至此，主私钥终于可用了。

4.  调用主私钥（扩展私钥）的 `Derive` 方法，开始衍生子密钥 `accountKey`。

        masterKey.Derive(accountKey, BIP32_HARDENED_KEY_LIMIT);

    注意，在 bip32 之后，采用硬化衍生子密钥。

    我们来看下 `Derive` 这个方法的执行流程：

    -   设置扩展私钥 `out` 参数的 `nDepth` 为 `nDepth` 加 1。主私钥的 `nDepth` 为 0，参见上面所讲。

    -   获取主公钥的 CKeyID。

             CKeyID id = key.GetPubKey().GetID();

    -   设置置扩展私钥 `out` 参数的 `nChild` 为参数 `_nChild` 的值，这里为 `0x80000000`，10进制为 2147483648。

    -   调用根私钥的 `Derive` 方法，开始衍生私钥。

            return key.Derive(out.key, out.chaincode, _nChild, chaincode);

        这个方法内部执行流程如下：

        -   生成一个向量。

                std::vector<unsigned char, secure_allocator<unsigned char>> vout(64);

        -   把变量 `nChild` 向右移动 31位，如果所得等于 0，那么调用 `GetPubKey` 方法，获得私钥对应的公钥，然后调用 `BIP32Hash` 方法，填充变量 `vout`。在 `BIP32Hash` 方法中使用 `CHMAC_SHA512` 方法，根据链码参数和子密钥数量来填充 `vout` 变量。如果右移所得不等于0，同样调用 `BIP32Hash` 方法，填充变量 `vout`。

                if ((nChild >> 31) == 0) {
                    CPubKey pubkey = GetPubKey();
                    assert(pubkey.size() == CPubKey::COMPRESSED_PUBLIC_KEY_SIZE);
                    BIP32Hash(cc, nChild, *pubkey.begin(), pubkey.begin()+1, vout.data());
                } else {
                    assert(size() == 32);
                    BIP32Hash(cc, nChild, 0, begin(), vout.data());
                }

        -   把变量 `vout` 的内容拷贝到子链码中。

                memcpy(ccChild.begin(), vout.data()+32, 32);

        -   把私钥的内容拷贝到子私钥中。

                memcpy((unsigned char*)keyChild.begin(), begin(), 32);

        -   使用椭圆曲线算法真正初始化子私钥。

                bool ret = secp256k1_ec_privkey_tweak_add(secp256k1_context_sign, (unsigned char*)keyChild.begin(), vout.data());

        -   设置子私钥为压缩的，是否是有效的，并返回。

                keyChild.fCompressed = true;
                keyChild.fValid = ret;
                return ret;

5.  `accountKey` 私钥（扩展私钥）的 `Derive` 方法，开始衍生子密钥 `chainChildKey`。方法前面刚讲过，此处略过。

        accountKey.Derive(chainChildKey, BIP32_HARDENED_KEY_LIMIT+(internal ? 1 : 0));

6.  接下来衍生子私钥 `childKey`，与上面大致相同，可自行阅读。

        do {
            if (internal) {
                chainChildKey.Derive(childKey, hdChain.nInternalChainCounter | BIP32_HARDENED_KEY_LIMIT);
                metadata.hdKeypath = "m/0'/1'/" + std::to_string(hdChain.nInternalChainCounter) + "'";
                hdChain.nInternalChainCounter++;
            }
            else {
                chainChildKey.Derive(childKey, hdChain.nExternalChainCounter | BIP32_HARDENED_KEY_LIMIT);
                metadata.hdKeypath = "m/0'/0'/" + std::to_string(hdChain.nExternalChainCounter) + "'";
                hdChain.nExternalChainCounter++;
            }
        } while (HaveKey(childKey.key.GetPubKey().GetID()));  

7.  设置变量 `secret` 的值为子私钥的 `childKey`。

        secret = childKey.key;

8.  设置变量 `metadata` 的 HD 种子为当前 HD 链的的种子。

        metadata.hd_seed_id = hdChain.seed_id;

9.  更新 HD 链到数据库中。

        if (!batch.WriteHDChain(hdChain))
            throw std::runtime_error(std::string(__func__) + ": Writing HD chain model failed");


#   后记

由于本人水平所限，文中错误在所难免，欢迎您踊跃指出错误，在下感激不尽。我的微信联系方式：joepeak。

原创不易，尤其寒冬，欢迎赞助我一杯咖啡。

<div style="display: inline-flex;">
    <div>
        <img src="http://payee.szdyfjh.com/btcpay.png" alt="drawing" style="width: 200px;" />
        <p style="text-align: center;">比特币</p>
    </div>
    <div>
        <img src="http://payee.szdyfjh.com/wechatpay.jpg" alt="drawing" style="width: 200px;margin-left: 20px;"/>
        <p style="text-align: center;">微信</p>
    </div>
    <div>
        <img src="http://payee.szdyfjh.com/alipay.jpg" alt="drawing" style="width: 200px;margin-left: 20px;"/>
        <p style="text-align: center;">支付宝</p>
    </div>
</div>
