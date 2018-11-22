
#   写在前面

创建钱包和交易是比特币最重要的两方面，涉及到很多很多的内容，远非一篇文章能概括的完。上一篇从整体上讲解了钱包的创建流程，虽然已经很详细了，但还是漏掉了几个重要的方法，从本篇开始我们讲解这几个特别重要的方法。对钱包感兴趣的朋友一定不要错误。

#   GenerateNewSeed 生成新的私钥/公钥

这个方法主要用来生成私钥/公钥，在生成公钥后，调用钱包对象的 `SetHDSeed` 方法，根据公钥生成一个 `CHDChain` 对象，把公钥经过 SHA256、RIPEMD160 两次哈希返回后的 20个字节 180 位字符串作为它的 `seed_id` 属性，从而以后可以衍生更的扩展公钥。

    void CWallet::SetHDSeed(const CPubKey& seed)
    {
        LOCK(cs_wallet);
        CHDChain newHdChain;
        newHdChain.nVersion = CanSupportFeature(FEATURE_HD_SPLIT) ? CHDChain::VERSION_HD_CHAIN_SPLIT : CHDChain::VERSION_HD_BASE;
        newHdChain.seed_id = seed.GetID();
        SetHDChain(newHdChain, false);
    }


在创建钱包的过程中，本方法及下面的 `TopUpKeyPool` 方法，可能会被调用两次，一次在第一次创建钱包时，另一次在用户明确升级，且钱包支持 HD，但 HD 没启用的情况下。

下面，我们开始正式讲解这个方法。

1.  首先，生成并调用私钥的 `MakeNewKey` 来初始化私钥。

        CKey key;
        key.MakeNewKey(true);

2.  然后，调用 `DeriveNewSeed` 方法，生成并返回私钥对应的公钥。

        return DeriveNewSeed(key);

我们先来看 `MakeNewKey` 这个方法。这个方法比较简单，但是非常重要，因为正是这个方法，生成了真正意义上的私钥。

    void CKey::MakeNewKey(bool fCompressedIn) {
        do {
            GetStrongRandBytes(keydata.data(), keydata.size());
        } while (!Check(keydata.data()));
        fValid = true;
        fCompressed = fCompressedIn;
    }

`GetStrongRandBytes` 方法，正是生成私钥的过程。这个方法的执行逻辑如下：

1.  生成一个 SHA512对象和一个无符号字符数组。

        CSHA512 hasher;
        unsigned char buf[64];

2.  首先，调用 `RandAddSeedPerfmon` 方法，通过 OpenSSL's RNG 生成随机数，并进行 SHA512 哈希。

        RandAddSeedPerfmon();
        GetRandBytes(buf, 32);
        hasher.Write(buf, 32);

3.  然后，调用 `GetOSRand` 方法，通过 OS RNG 生成随机数，并进行 SHA512 哈希。

        GetOSRand(buf);
        hasher.Write(buf, 32);

4.  最后，如果 HW 可用，通过 HW 生成随机数，并进行 SHA512 哈希。

        if (GetHWRand(buf)) {
            hasher.Write(buf, 32);
        }

5.  接下来，合并并更新状态。

        {
            WAIT_LOCK(cs_rng_state, lock);
            hasher.Write(rng_state, sizeof(rng_state));
            hasher.Write((const unsigned char*)&rng_counter, sizeof(rng_counter));
            ++rng_counter;
            hasher.Finalize(buf);
            memcpy(rng_state, buf + 32, 32);
        }

6.  然后，把 `buf` 数组中的内容拷贝到输出参数中，并清空数组中的内容。

        memcpy(out, buf, num);
        memory_cleanse(buf, 64);

`Check` 方法内部通过调用 `secp256k1_ec_seckey_verify` 方法，检查私钥的数据是否满足要求，后者正是椭圆曲线算法相关的。

接下来，我们来看 `DeriveNewSeed` 方法是如何生成公钥的。方法执行逻辑如下：

1.  生成当前时间，并用当前时间初始化蜜钥元数据。

        int64_t nCreationTime = GetTime();
        CKeyMetadata metadata(nCreationTime);

2.  调用私钥的 `GetPubKey` 方法，生成公钥。

    方法内部正是通过椭圆曲线算法来生成私钥对应的公钥。代码如下，有兴趣的读者可以自行研究。

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

3.  执行 `assert(key.VerifyPubKey(seed));` 方法，验证公钥确实有私钥相匹配。

4.  设置蜜钥元数据的两个属性。

        metadata.hdKeypath     = "s";
        metadata.hd_seed_id = seed.GetID();

    `GetID` 方法，用公钥的数据经过 SHA256、RIPEMD160 两次哈希后生成的字符串来生成一个 `CKeyID` 对象。

5.  把密钥元数据保存在 `mapKeyMetadata` 集合中。

        mapKeyMetadata[seed.GetID()] = metadata;

6.  调用 `AddKeyPubKey` 方法，把私钥/公钥及元数据保存到数据库。如果出错，抛出一个异常。

        if (!AddKeyPubKey(key, seed))
            throw std::runtime_error(std::string(__func__) + ": AddKeyPubKey failed");

    `AddKeyPubKey` 方法生成一个访问数据库的对象，然后调用钱包对象的 `AddKeyPubKeyWithDB` 方法来保存密钥数据。后者的执行流程如下：

    -   如果变量 `encrypted_batch` 为空，那么设置变量 `needsDB` 为真。如果后者为真，那么设置 `encrypted_batch` 属性为参数指定的对象。

            bool needsDB = !encrypted_batch;
            if (needsDB) {
                encrypted_batch = &batch;
            }

    -   调用 `CCryptoKeyStore::AddKeyPubKey` 方法，保存私钥和公钥。如果不成功，进一步，如果变量 `needsDB` 为真，则设置变量 `encrypted_batch` 为假。

            if (!CCryptoKeyStore::AddKeyPubKey(secret, pubkey)) {
                if (needsDB) encrypted_batch = nullptr;
                return false;
            }

        `CCryptoKeyStore::AddKeyPubKey` 方法内部执行逻辑如下：

        -   如果没有加密，那么调用 `CBasicKeyStore::AddKeyPubKey` 来保存私钥和公钥，然后返回。

                if (!IsCrypted()) {
                    return CBasicKeyStore::AddKeyPubKey(key, pubkey);
                }

            `IsCrypted` 方法，初始化时是没有加密的，只有在执行加锁、解锁、添加加密密钥的情况下才会是加密的。当前情景是没有加密的，所以执行基类的方法来保存私钥和公钥。

            `CBasicKeyStore::AddKeyPubKey` 方法，首先把私钥保存在 `mapKeys` 集合中，然后调用 `ImplicitlyLearnRelatedKeyScripts` 方法来处理脚本，因为公钥默认是压缩的，所以在方法内会生成脚本，并把脚本保存在 `mapScripts` 集合中。

                bool CBasicKeyStore::AddKeyPubKey(const CKey& key, const CPubKey &pubkey)
                {
                    LOCK(cs_KeyStore);
                    mapKeys[pubkey.GetID()] = key;
                    ImplicitlyLearnRelatedKeyScripts(pubkey);
                    return true;
                }

        -   如果是锁定的，那么返回假。

                if (IsLocked()) {
                    return false;
                }

            `IsLocked` 方法，首先调用 `IsCrypted` 方法，确定是否是加密的，如果不是加密的，则直接返回假；否则，把 `vMasterKey` 集合清空。

                bool CCryptoKeyStore::IsLocked() const
                {
                    if (!IsCrypted()) {
                        return false;
                    }
                    LOCK(cs_KeyStore);
                    return vMasterKey.empty();
                }

        -   接下来，生成加密私钥。

                std::vector<unsigned char> vchCryptedSecret;
                CKeyingMaterial vchSecret(key.begin(), key.end());
                if (!EncryptSecret(vMasterKey, vchSecret, pubkey.GetHash(), vchCryptedSecret)) {
                    return false;
                }

        -   最后，调用 `AddCryptedKey` 方法，保存加密私钥，并返回

                if (!AddCryptedKey(pubkey, vchCryptedSecret)) {
                    return false;
                }
                return true;

            `AddCryptedKey` 方法，首先确定是否是加密的，如果没有加密则返回假。接下来，把私钥保存在 `mapCryptedKeys` 集合中，然后调用 `ImplicitlyLearnRelatedKeyScripts` 方法来处理脚本，因为公钥默认是压缩的，所以在方法内会生成脚本，并把脚本保存在 `mapScripts` 集合中。

                bool CCryptoKeyStore::AddCryptedKey(const CPubKey &vchPubKey, const std::vector<unsigned char> &vchCryptedSecret)
                {
                    LOCK(cs_KeyStore);
                    if (!SetCrypted()) {
                        return false;
                    }
                    mapCryptedKeys[vchPubKey.GetID()] = make_pair(vchPubKey, vchCryptedSecret);
                    ImplicitlyLearnRelatedKeyScripts(vchPubKey);
                    return true;
                }

    -   如果变量 `needsDB` 为真，则设置变量 `encrypted_batch` 为空指针。

    -   调用 `GetScriptForDestination` 方法，获得公钥对应的脚本。

            CScript script;
            script = GetScriptForDestination(pubkey.GetID());

    -   调用 `HaveWatchOnly` 方法，检查 `setWatchOnly` 集合中是否有这个脚本。如果有，则调用 `RemoveWatchOnly` 方法，清除公钥对应的脚本。

            if (HaveWatchOnly(script)) {
                RemoveWatchOnly(script);
            }

        `RemoveWatchOnly` 方法执行逻辑如下：

        -   调用 `CCryptoKeyStore::RemoveWatchOnly` 方法来移除脚本。方法内部执行逻辑如下：从 `setWatchOnly` 集合中移除对应的脚本；然后，调用 `ExtractPubKey` 方法，通过脚本来解析出公钥，如果可以得到公钥，则从 `mapWatchKeys` 集合中移除对应的公钥；最后，返回真。

        -   调用数据库访问对象的 `EraseWatchOnly` 方法，从数据库中移除 `watchmeta` 和 `watchs` 对应的数据。

        -   返回真。

    -   调用 `GetScriptForRawPubKey` 方法，从原始公钥获得脚本。

            script = GetScriptForRawPubKey(pubkey);

    -   调用 `HaveWatchOnly` 方法，检查 `setWatchOnly` 集合中是否有这个脚本。如果有，则调用 `RemoveWatchOnly` 方法，清除公钥对应的脚本。

        方法执行逻辑如上所述，这里不讲。

    -   如果不是加密的，则调用数据库的访问对象的 `WriteKey` 方法，把私钥和公钥及密钥元数据写入数据库。

        在这个方法中，首先，以 `keymeta` 为键，保存对应的元数据；然后，把公钥/私钥保存到向量中，以 `key` 为键，保存对应的公钥/私钥。

    -   最后，返回真。

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
