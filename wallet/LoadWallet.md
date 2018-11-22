#   写在前面

创建钱包和交易是比特币最重要的两方面，涉及到很多很多的内容，远非一篇文章能概括的完。上一篇从整体上讲解了钱包的创建流程，虽然已经很详细了，但还是漏掉了几个重要的方法，从本篇开始我们讲解这几个特别重要的方法。对钱包感兴趣的朋友一定不要错误。

#   WalletBatch::LoadWallet 加载钱包

本方法，从数据库中加载钱包的相关数据。方法的执行逻辑如下：

1.  首先，初始化几个变量。

        CWalletScanState wss;
        bool fNoncriticalErrors = false;
        DBErrors result = DBErrors::LOAD_OK;

2.  从数据库中读取钱包的最小版本。如果最小版本大于当前的最新版本，则返回数据库异常；否则，调用钱包对象的 `LoadMinVersion` 方法，设置钱包的版本相关属性，`nWalletVersion` 属性为数据库读取取的 `nMinVersion`，`nWalletMaxVersion` 属性为 `nWalletMaxVersion` 与这个值的较大者。

        int nMinVersion = 0;
        if (m_batch.Read((std::string)"minversion", nMinVersion))
        {
            if (nMinVersion > FEATURE_LATEST)
                return DBErrors::TOO_NEW;
            pwallet->LoadMinVersion(nMinVersion);
        }

3.  获取数据库游标，如果出错，则返回数据库游标错误。

        Dbc* pcursor = m_batch.GetCursor();
        if (!pcursor)
        {
            pwallet->WalletLogPrintf("Error getting wallet database cursor\n");
            return DBErrors::CORRUPT;
        }

4.  进放 ` while (true){ ... }` 循环读取数据库中的所有数据。具体处理如下：

    -   调用 `ReadAtCursor` 方法从游标中读取对应的数据。如果没有读取到数据，则退出循环。如果出现错误，则调用钱包对象的 `WalletLogPrintf` 方法打印，然后返回数据库错误。

            CDataStream ssKey(SER_DISK, CLIENT_VERSION);
            CDataStream ssValue(SER_DISK, CLIENT_VERSION);
            int ret = m_batch.ReadAtCursor(pcursor, ssKey, ssValue);
            if (ret == DB_NOTFOUND)
                break;
            else if (ret != 0)
            {
                pwallet->WalletLogPrintf("Error reading next record from wallet database\n");
                return DBErrors::CORRUPT;
            }

    -   调用 `ReadKeyValue` 方法，读取游标中当前的内容。如果读取出错，则根据错误进行相应的处理，具体不细说。如果读取数据成功，其内部根据读取到的数据进行具体处理如下：

        -   如果当前的的 Key 是 `name`，那么从流中读取对应的地址到变量 `strAddress`中，调用 `DecodeDestination` 方法解码这个地址，然后设置钱包对象 `mapAddressBook` 集合对应的 `CAddressBookData` 对象的 `name` 属性为流中读取到的名字值。

                if (strType == "name")
                {
                    std::string strAddress;
                    ssKey >> strAddress;
                    ssValue >> pwallet->mapAddressBook[DecodeDestination(strAddress)].name;
                }

        -   否则，如果前的 Key 是 `purpose` ，那么从流中读取对应的地址到变量 `strAddress`中，调用 `DecodeDestination` 方法解码这个地址，然后设置钱包对象 `mapAddressBook` 集合对应的 `CAddressBookData` 对象的 `purpose` 属性为流中读取到的名字值。

                else if (strType == "purpose")
                {
                    std::string strAddress;
                    ssKey >> strAddress;
                    ssValue >> pwallet->mapAddressBook[DecodeDestination(strAddress)].purpose;
                }

        -   否则，如果当前的 Key 是 `tx` ，即交易，那么处理如下。

            从流中读取交易哈希和钱包交易对象，然后调用 `CheckTransaction` 方法，检查读取的钱包交易对象，如果出错，则返回假。

                uint256 hash;
                ssKey >> hash;
                CWalletTx wtx(nullptr /* pwallet */, MakeTransactionRef());
                ssValue >> wtx;
                CValidationState state;
                if (!(CheckTransaction(*wtx.tx, state) && (wtx.GetHash() == hash) && state.IsValid()))
                    return false;

            如果钱包交易对象的 `fTimeReceivedIsTxTime` 属性大于等于 31404，且小于等于 31703，因为撤销序列化在 31600 中进行了变更，所以进行下面的序列化处理。

                if (31404 <= wtx.fTimeReceivedIsTxTime && wtx.fTimeReceivedIsTxTime <= 31703)
                {
                    if (!ssValue.empty())
                    {
                        char fTmp;
                        char fUnused;
                        std::string unused_string;
                        ssValue >> fTmp >> fUnused >> unused_string;
                        strErr = strprintf("LoadWallet() upgrading tx ver=%d %d %s", wtx.fTimeReceivedIsTxTime, fTmp, hash.ToString());
                        wtx.fTimeReceivedIsTxTime = fTmp;
                    }
                    else
                    {
                        strErr = strprintf("LoadWallet() repairing tx ver=%d %s", wtx.fTimeReceivedIsTxTime, hash.ToString());
                        wtx.fTimeReceivedIsTxTime = 0;
                    }
                    wss.vWalletUpgrade.push_back(hash);
                }

            如果钱包交易对象的 `nOrderPos` 为-1，那么设置钱包扫描状态对象的 `fAnyUnordered` 为真。

                if (wtx.nOrderPos == -1)
                    wss.fAnyUnordered = true;

            调用钱包对象的 `LoadToWallet`，加载数据库中读到的钱包交易对象到钱包中。

        -   否则，如果当前的 Key 是 `watchs` ，那么从流中读取对应的序列化脚本，并读取其对应的值。如果其值等于1，那么调用钱包对象的 `LoadWatchOnly` 方法，加载一个只读的脚本。

                wss.nWatchKeys++;
                CScript script;
                ssKey >> script;
                char fYes;
                ssValue >> fYes;
                if (fYes == '1')
                    pwallet->LoadWatchOnly(script);

        -   否则，如果当前的 Key 是 `key` 或者 `wkey`，那么处理如下。

            从流中读取对应的公钥，如果 Key 是 `key`，那么从流中读取出对应的私钥，否则从流中读取对应的钱包私钥，从中取得对应的私钥。

                CPubKey vchPubKey;
                ssKey >> vchPubKey;
                if (!vchPubKey.IsValid())
                {
                    strErr = "Error reading wallet database: CPubKey corrupt";
                    return false;
                }
                CKey key;
                CPrivKey pkey;
                uint256 hash;
                if (strType == "key")
                {
                    wss.nKeys++;
                    ssValue >> pkey;
                } else {
                    CWalletKey wkey;
                    ssValue >> wkey;
                    pkey = wkey.vchPrivKey;
                }

            从流中读取对应的哈希值。

                ssValue >> hash;

            如果哈希不空，则处理如下：

                if (!hash.IsNull())
                {
                    std::vector<unsigned char> vchKey;
                    vchKey.reserve(vchPubKey.size() + pkey.size());
                    vchKey.insert(vchKey.end(), vchPubKey.begin(), vchPubKey.end());
                    vchKey.insert(vchKey.end(), pkey.begin(), pkey.end());

                    if (Hash(vchKey.begin(), vchKey.end()) != hash)
                    {
                        strErr = "Error reading wallet database: CPubKey/CPrivKey corrupt";
                        return false;
                    }

                    fSkipCheck = true;
                }

            调用 `CKey` 对象的 `Load` 方法加载密钥。

                if (!key.Load(pkey, vchPubKey, fSkipCheck))
                {
                    strErr = "Error reading wallet database: CPrivKey corrupt";
                    return false;
                }

            调用钱包对象的 `LoadKey` 方法，加载密钥。

                if (!pwallet->LoadKey(key, vchPubKey))
                {
                    strErr = "Error reading wallet database: LoadKey failed";
                    return false;
                }

        -   否则，如果当前的 Key 是 `mkey`，则从流中读取对应的主密钥，并保存在钱包 `mapMasterKeys` 对应的位置。

                else if (strType == "mkey")
                {
                    unsigned int nID;
                    ssKey >> nID;
                    CMasterKey kMasterKey;
                    ssValue >> kMasterKey;
                    if(pwallet->mapMasterKeys.count(nID) != 0)
                    {
                        strErr = strprintf("Error reading wallet database: duplicate CMasterKey id %u", nID);
                        return false;
                    }
                    pwallet->mapMasterKeys[nID] = kMasterKey;
                    if (pwallet->nMasterKeyMaxID < nID)
                        pwallet->nMasterKeyMaxID = nID;
                }

        -   否则，如果当前的 Key 是 `ckey`，则从流中读取对应的公钥、私钥，并调用钱包对象的 `LoadCryptedKey` 方法，加载加密的密钥。

                else if (strType == "ckey")
                {
                    CPubKey vchPubKey;
                    ssKey >> vchPubKey;
                    if (!vchPubKey.IsValid())
                    {
                        strErr = "Error reading wallet database: CPubKey corrupt";
                        return false;
                    }
                    std::vector<unsigned char> vchPrivKey;
                    ssValue >> vchPrivKey;
                    wss.nCKeys++;

                    if (!pwallet->LoadCryptedKey(vchPubKey, vchPrivKey))
                    {
                        strErr = "Error reading wallet database: LoadCryptedKey failed";
                        return false;
                    }
                    wss.fIsEncrypted = true;
                }

        -   否则，如果当前的 Key 是 `keymeta`，那么从流中读取对应的公钥及其元数据，调用钱包对象的 `LoadKeyMetadata` 方法，加载密钥元数据。

                else if (strType == "keymeta")
                {
                    CPubKey vchPubKey;
                    ssKey >> vchPubKey;
                    CKeyMetadata keyMeta;
                    ssValue >> keyMeta;
                    wss.nKeyMeta++;
                    pwallet->LoadKeyMetadata(vchPubKey.GetID(), keyMeta);
                }

       -    否则，如果当前的 Key 是 `watchmeta`，那么从流中读取脚本和对应的元数据，并调用钱包对象的 `LoadScriptMetadata` 方法，加载脚本的元数据。

                else if (strType == "watchmeta")
                {
                    CScript script;
                    ssKey >> script;
                    CKeyMetadata keyMeta;
                    ssValue >> keyMeta;
                    wss.nKeyMeta++;
                    pwallet->LoadScriptMetadata(CScriptID(script), keyMeta);
                }

        -   否则，如果当前的 Key 是 `defaultkey`，那么从流中读取对应的公钥。

                else if (strType == "defaultkey")
                {
                    CPubKey vchPubKey;
                    ssValue >> vchPubKey;
                    if (!vchPubKey.IsValid()) {
                        strErr = "Error reading wallet database: Default Key corrupt";
                        return false;
                    }
                }

        -   否则，如果当前的 Key 是 `pool`，那么从流中读取对应的密钥池，并调用钱包对象的 `LoadKeyPool` 方法，加载密钥池。

                else if (strType == "pool")
                {
                    int64_t nIndex;
                    ssKey >> nIndex;
                    CKeyPool keypool;
                    ssValue >> keypool;
                    pwallet->LoadKeyPool(nIndex, keypool);
                }

        -   否则，如果当前的 Key 是 `version`，那么从流中读取对应的版本到钱包扫描状态对象的 `nFileVersion` 属性中。如果这个值等于 10300，那么设置钱包对象的这个属性为 300。

                else if (strType == "version")
                {
                    ssValue >> wss.nFileVersion;
                    if (wss.nFileVersion == 10300)
                        wss.nFileVersion = 300;
                }

        -   否则，如果当前的 Key 是 `cscript`，那么从流中读取对应的脚本，并调用钱包对象的 `LoadCScript` 方法加载脚本。

                else if (strType == "cscript")
                {
                    uint160 hash;
                    ssKey >> hash;
                    CScript script;
                    ssValue >> script;
                    if (!pwallet->LoadCScript(script))
                    {
                        strErr = "Error reading wallet database: LoadCScript failed";
                        return false;
                    }
                }

        -   否则，如果当前的 Key 是 `orderposnext`，那么从流中读取对应的值到钱包对象的 `nOrderPosNext` 属性中。

                else if (strType == "orderposnext")
                {
                    ssValue >> pwallet->nOrderPosNext;
                }

        -   否则，如果当前的 Key 是 `destdata`，那么从流中读取对应的数据，并调用钱包对象的 `LoadDestData` 方法，处理这些数据。

                else if (strType == "destdata")
                {
                    std::string strAddress, strKey, strValue;
                    ssKey >> strAddress;
                    ssKey >> strKey;
                    ssValue >> strValue;
                    pwallet->LoadDestData(DecodeDestination(strAddress), strKey, strValue);
                }

        -   否则，如果当前的 Key 是 `hdchain`，那么从流中读取 HD 链，并调用钱包对象的 `SetHDChain`方法，设置钱包的 HD 链中。

                else if (strType == "hdchain")
                {
                    CHDChain chain;
                    ssValue >> chain;
                    pwallet->SetHDChain(chain, true);
                }

        -   否则，如果当前的 Key 是 ``，那么调用钱包的 `` 方法，设置其标志。

                else if (strType == "flags") {
                    uint64_t flags;
                    ssValue >> flags;
                    if (!pwallet->SetWalletFlags(flags, true)) {
                        strErr = "Error reading wallet database: Unknown non-tolerable wallet flags found";
                        return false;
                    }
                }

        -   否则，如果当前的 Key 不是 `bestblock`、`bestblock_nomerkle`、`minversion`、`acentry` ，则钱包扫描状态对象的 `m_unknown_records` 未知道记录属性加1。

5.  返回真。


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
