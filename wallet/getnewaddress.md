#  生成地址

如果有人想发送比特币给你，或者你从别人那里买几个比特币，就要把地址给对方，对方才能把币打到你指定的地址上。那么，如何才能拥有一个地址呢，下面我们就来讲讲这个问题。

比特币核心提供了很多 RPC 来供客户端调用，其中一个就是我们这里要讲的 `getnewaddress` 生成一个新的地址，通过这个 RPC ，我们就可以生成一个新的地址，有了这个地址，别人就可以给我们转账了。

`getnewaddress` RPC 可以接收两个参数，第一个地址的标签，第二个是地址的类型。如果没有提供标签，那么默认的标签就是空，地址的类型当前支持：legacy、p2sh-segwit、bech32，默认类型由 `-addresstype` 参数指定，当前为 p2sh-segwit。


如果我们想看下这个 RPC 的帮助文档，可以执行如下的命令：

	./src/bitcoin-cli -regtest help getnewaddress

就会显示帮助信息

这个　RPC 对应的方法实现位于 `src/wallet/rpcwallet.cpp` 文件，方法名称就是 RPC 名称，下面我们来看这个方法。

##  生成地址流程

1.  根据请求参数获得对应的钱包。

        std::shared_ptr<CWallet> const wallet = GetWalletForJSONRPCRequest(request);
        CWallet* const pwallet = wallet.get();

	`GetWalletForJSONRPCRequest` 方法内部实现如下：

    -   调用 `GetWalletNameFromJSONRPCRequest` 方法，从请求对象中取得钱包的名字，如果用户指定了钱包名字，那么把钱包名字保存在参数 `wallet_name` 上，并返回真，否则返回假。

    -   如果可以获得用户指定的钱包名称，则调用 `GetWallet` 方法，从钱包集合 `vpwallets` 中取得指定的钱包，然后返回钱包。

    -   如果用户没有指定钱包或指定的钱包不存在，那么调用 `GetWallets` 方法，返回钱包集合 `vpwallets`。如果钱包集合中只有一个钱包，或者在用户指定了帮助的情况下，至少有一个以上的钱包，那么返回第一个钱包，即默认的钱包。默认钱包在系统启动时候创建的。

2.	接下来，要确保钱包可用。如果钱包不可用，则直接 `NullUniValue` 对象。

            if (!EnsureWalletIsAvailable(pwallet, request.fHelp)) {
    	        return NullUniValue;
    	    }

3.  如果指定了 `help` 参数或请求参数数量多于2个，则显示钱包的帮助信息。

4.	检查钱包是否设置了禁止私钥，即钱包是只读的 `watch-only/pubkeys`。如果是，则抛出异常。

    	    if (pwallet->IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS)) {
    	        throw JSONRPCError(RPC_WALLET_ERROR, "Error: Private keys are disabled for this wallet");
    	    }

5.	如果指定了标签，则调用 `LabelFromValue` 方法，检查标签，确保其不是 `*`。如果是，则抛出异常。

    	    std::string label;
    	    if (!request.params[0].isNull())
    	        label = LabelFromValue(request.params[0]);

6.	如果指定了地址类型，则调用 `ParseOutputType` 方法，检查地址类型，确保其是 `legacy`、`p2sh-segwit`、`bech32` 之一，如果不指定则默认是 `p2sh-segwit`，并把地址类型保存在 `output_type` 变量中。

    	    OutputType output_type = pwallet->m_default_address_type;
    	    if (!request.params[1].isNull()) {
    	        if (!ParseOutputType(request.params[1].get_str(), output_type)) {
    	            throw JSONRPCError(RPC_INVALID_ADDRESS_OR_KEY, strprintf("Unknown address type '%s'", request.params[1].get_str()));
    	        }
    	    }

7.	如果钱包没有被锁定，则调用 `TopUpKeyPool` 方法填充密钥池。

    	    if (!pwallet->IsLocked()) {
    	        pwallet->TopUpKeyPool();
    	    }

    `TopUpKeyPool` 填充密钥这个方法，我们前面已经讲过，这里只简单解释下，不做详细分析。因为在衍生子钥的过程中，`setExternalKeyPool`、`setInternalKeyPool` 已经完全填充完了，所以导致 `missingExternal`、`missingInternal` 两个变量都为 0，从而不会重新再次衍生子密钥，所以实际上本方法在这里基本没有执行，而直接返回真。

8.  调用钱包的 `GetKeyFromPool` 方法，从密钥池中生成一个公钥。如果不能生成，则抛出异常。

        CPubKey newKey;
        if (!pwallet->GetKeyFromPool(newKey)) {
            throw JSONRPCError(RPC_WALLET_KEYPOOL_RAN_OUT, "Error: Keypool ran out, please call keypoolrefill first");
        }

    `GetKeyFromPool` 方法，我们在下面详细讲解，此处略过。

9.  调用钱包对象的 `LearnRelatedScripts` 方法，对公钥的脚本进行处理。

    方法内部执行如下：

    如果公钥是压缩的，并且地址类型是 `p2sh-segwit`，或者 `bech32`，那么：

    -   调用 `WitnessV0KeyHash` 方法，生成 `WitnessV0KeyHash` 对象。

            CTxDestination witdest = WitnessV0KeyHash(key.GetID());

    -   调用 `GetScriptForDestination` 方法，获取对应的脚本。

            CScript witprog = GetScriptForDestination(witdest);

        `GetScriptForDestination` 方法内部调用 `boost::apply_visitor(CScriptVisitor(&script), dest)`，以访问者模式来根据不同的 id，获取其对应的脚本对象。

        `CScriptVisitor` 对象继承自 `boost::static_visitor` 对象，实现了访问者模式，并通过重载 `()` 操作符来定义不同类型的 id。

        -   如果目标参数类型是 `CNoDestination`，则调用脚本对象的 `script` 方法，清除脚本内容。

        -   如果目标参数类型是 `CKeyID`，则：首先调用脚本对象的 `script` 方法，清除脚本内容；然后，初始化脚本 `*script << OP_DUP << OP_HASH160 << ToByteVector(keyID) << OP_EQUALVERIFY << OP_CHECKSIG`。

        -   如果目标参数类型是 `CScriptID`，则：首先调用脚本对象的 `script` 方法，清除脚本内容；然后，初始化脚本 `*script << OP_HASH160 << ToByteVector(scriptID) << OP_EQUAL`。

        -   如果目标参数类型是 `WitnessV0KeyHash`，则：首先调用脚本对象的 `script` 方法，清除脚本内容；然后，初始化脚本 ` *script << OP_0 << ToByteVector(id)`。

        -   如果目标参数类型是 `WitnessV0ScriptHash`，则：首先调用脚本对象的 `script` 方法，清除脚本内容；然后，初始化脚本 `*script << OP_0 << ToByteVector(id)`。

        -   如果目标参数类型是 `WitnessUnknown`，则：首先调用脚本对象的 `script` 方法，清除脚本内容；然后，初始化脚本 `*script << CScript::EncodeOP_N(id.version) << std::vector<unsigned char>(id.program, id.program + id.length)`。

    -   调用 `AddCScript` 方法，保存脚本对象。

        `AddCScript` 方法，首先调用 `CCryptoKeyStore::AddCScript` 方法，把脚本保存到 key store 的 `mapScripts` 集合中；然后，调用数据库访问对象的 `WriteCScript` 方法，以 `cscript` 为键把脚本保存到数据库中。

            bool CWallet::AddCScript(const CScript& redeemScript)
            {
                if (!CCryptoKeyStore::AddCScript(redeemScript))
                    return false;
                return WalletBatch(*database).WriteCScript(Hash160(redeemScript), redeemScript);
            }

10. 调用 `GetDestinationForKey` 方法，获取目的地 `CTxDestination` 对象。

    `CTxDestination` 是一个具有特定目标的交易输出脚本模板。

    定义如下：`typedef boost::variant<CNoDestination, CKeyID, CScriptID, WitnessV0ScriptHash, WitnessV0KeyHash, WitnessUnknown> CTxDestination`，可能是以下几种类型之一：

    -   CNoDestination

        没有目的地设置

    -   CKeyID

        P2PKH 目的

    -   CScriptID

        P2SH 目的

    -   WitnessV0ScriptHash

        P2WSH 目的

    -   WitnessV0KeyHash

        P2WPKH 目的

    -   WitnessUnknown

        未知目的 P2W???

    `GetDestinationForKey` 方法，使用 `case` 表达式来根据不同的地址类型，生成不同的目的 `CTxDestination`。

    -   如果地址类型是 `legacy`，则直接返回公钥的 `KeyID`。内部把公钥的数据通过 SHA256 和 RIPEMD160 双重哈希之后，构造一个 `CKeyID` 对象。

    -   如果地址类型是 `p2sh-segwit`，或 `bech32`，则处理如下：

        -   如果公钥不是压缩的，处理方法 `legacy`。

                if (!key.IsCompressed()) return key.GetID();

        -   否则，生成 `WitnessV0KeyHash` 对象，然后调用 `GetScriptForDestination` 方法，获取对应的脚本，最后根据不同的地址类型生成的目的。

                CTxDestination witdest = WitnessV0KeyHash(key.GetID());
                CScript witprog = GetScriptForDestination(witdest);
                if (type == OutputType::P2SH_SEGWIT) {
                    return CScriptID(witprog);
                } else {
                    return witdest;
                }

            对于默认的、不传地址类型的情况，就会返回 `CScriptID` 类型的 `CTxDestination`，这个返回值在下面两步中都会用到。

11. 调用钱包对象的 `SetAddressBook` 方法，来保存公钥地址。

        pwallet->SetAddressBook(dest, label, "receive");

    `SetAddressBook` 方法执行如下：

    -   从 `mapAddressBook` 集合中，取得对应的目的数据。

            std::map<CTxDestination, CAddressBookData>::iterator mi = mapAddressBook.find(address);

    -   根据集合中是否有对应的数据设置变量是否为更新。

            fUpdated = mi != mapAddressBook.end();

    -   把标签保存为地址对应的 `CAddressBookData` 的 `name` 属性。

            mapAddressBook[address].name = strName;

    -   如果参数 `strPurpose` 不空，则更新地址对应的 `CAddressBookData` 的 `purpose` 属性。

            if (!strPurpose.empty()) mapAddressBook[address].purpose = strPurpose;

    -   调用数据库访问对象的 `WritePurpose` 方法，保存参数 `strPurpose` 到数据库中。

            if (!strPurpose.empty() && !WalletBatch(*database).WritePurpose(EncodeDestination(address), strPurpose))
                return false;

    -   调用数据库访问对象的 `WritePurpose` 方法，保存地址到数据库中。

            WalletBatch(*database).WriteName(EncodeDestination(address), strName);

        `strName` 为用户提供的标签。`EncodeDestination` 方法，我们在下一步讲解。

12. 调用 `EncodeDestination` 方法，解码目的地址，并返回其结果。

    `EncodeDestination` 方法同样采用了访问者模式 `return boost::apply_visitor(DestinationEncoder(Params()), dest)`。`DestinationEncoder` 类继承了 `boost::static_visitor`，实现了访问者模式，通过重载 `()` 操作符来定义不同类型的 id。

    与前面相对应，这个方法会处理 `CKeyID`、`CScriptID`、`WitnessV0KeyHash`、`WitnessV0ScriptHash`、`WitnessUnknown` 这几种不同情况。对于我们的默认情况来说，目的地址类型为 `CScriptID`，下面我们就看下这种情况的处理，其他情况可自行阅读。

    -   调用当前网络参数的 `Base58Prefix` 方法，返回脚本前缀。

            std::vector<unsigned char> data = m_params.Base58Prefix(CChainParams::SCRIPT_ADDRESS);

        对于主网络前缀是 5，测试网络是 196，回归测试网络是 196。
        
    -   把当前 20 个字节的数据加在前缀后面形成 21 个字节的字符串。

            data.insert(data.end(), id.begin(), id.end());

    -   调用 `EncodeBase58Check` 方法，编码成 Base58Check 格式，并返回其值。

            return EncodeBase58Check(data);

        下面，我们来看下 `EncodeBase58Check` 这个方法的处理。它的内部执行流程如下：用 21 个字节的字符串生成一个向量，同时调用 `Hash` 方法，生成一个 32 字节长的哈希字符串；然后把其最前面的 4个字节作为校验各加在 21 个字节的向量尾部，从而生成一个长度为 25个字节的字符串；最后，调用 `EncodeBase58` 方法，进行 Base58 编码。

            std::vector<unsigned char> vch(vchIn);
            uint256 hash = Hash(vch.begin(), vch.end());
            vch.insert(vch.end(), (unsigned char*)&hash, (unsigned char*)&hash + 4);
            return EncodeBase58(vch);

        在 `Hash` 这个方法中，使用了双重 SHA256 哈希算法。`EncodeBase58` 这个方法，读者可以自行阅读，这里不再展开。


##  GetKeyFromPool 从密钥池中获取公钥

本方法从密钥池中生成一个公钥。第一个参数为公钥的引用，第二个参数 `internal`，默认为假。

内部逻辑如下：

1.  如果钱包禁止私钥，则返回假。

        if (IsWalletFlagSet(WALLET_FLAG_DISABLE_PRIVATE_KEYS)) {
            return false;
        }

2.  调用 `ReserveKeyFromKeyPool` 方法，从密钥池中取出一个密钥并获取其公钥。如果不成功，则生成数据库访问对象，然后调用 `GenerateNewKey` 方法，生成一个公钥。

        if (!ReserveKeyFromKeyPool(nIndex, keypool, internal)) {
            if (IsLocked()) return false;
            WalletBatch batch(*database);
            result = GenerateNewKey(batch, internal);
            return true;
        }

    `GenerateNewKey` 这个方法，在创建钱包过程中，我们已经重点分析，这里不浪费口舌，我们重点看下 `ReserveKeyFromKeyPool` 方法。这个方法的执行流程如下：

    -   生成一个公钥，并设置为密钥池的 `vchPubKey` 属性。

            nIndex = -1;
            keypool.vchPubKey = CPubKey();

    -   如果钱包没有被锁，则填充密钥池。

            if (!IsLocked())
                TopUpKeyPool();

        `TopUpKeyPool` 这个方法，我们也讲过，这里直接略过。

    -   如果钱包启用了 HD，并且可以支持 HD 分割，并且参数 `fRequestedInternal` 为真，则设置变量 `fReturningInternal` 为真。在调用本方法时，这个参数没有指定，而默认为假，所以变量 `fRequestedInternal` 设置假。

            bool fReturningInternal = IsHDEnabled() && CanSupportFeature(FEATURE_HD_SPLIT) && fRequestedInternal;

    -   根据集合 `set_pre_split_keypool` 是否为空，设置变量 `use_split_keypool` 的值。因为这里 `use_split_keypool` 集合为空，所以变量 `use_split_keypool` 为真。

            bool use_split_keypool = set_pre_split_keypool.empty();

    -   根据变量 `use_split_keypool`、`fReturningInternal` 确定从哪个集合中获取密钥池对象。根据上面分析，我们最终会从 `setExternalKeyPool` 集合中取数据。

            std::set<int64_t>& setKeyPool = use_split_keypool ? (fReturningInternal ? setInternalKeyPool : setExternalKeyPool) : set_pre_split_keypool;

    -   如果要数据的集合为为空，则返回假。

            if (setKeyPool.empty()) {
                return false;
            }

    -   生成数据库访问对象。
    
            WalletBatch batch(*database);

    -   从 `setKeyPool` 取得其第一个元素，并从集合中删除它。

            auto it = setKeyPool.begin();
            nIndex = *it;
            setKeyPool.erase(it);

    -   从数据库取得索引对应的密钥池。如果失败，则抛出异常。

            if (!batch.ReadPool(nIndex, keypool)) {
                throw std::runtime_error(std::string(__func__) + ": read failed");
            }

    -   从密钥池中取得公钥对应的 ID，并且检测其是否在 `mapKeys`、或 `mapCryptedKeys` 集合之一，如果不在，则抛出异常。我们在创建钱包过程时候讲过，生成的私钥根据是否加密会保存在这两个集合之一。

            if (!HaveKey(keypool.vchPubKey.GetID())) {
                throw std::runtime_error(std::string(__func__) + ": unknown key in key pool");
            }

    -   如果变量 `use_split_keypool` 为真，并且密钥池的 `fInternal` 属性不等于变量 `fReturningInternal`，那么抛出异常。

            if (use_split_keypool && keypool.fInternal != fReturningInternal) {
                throw std::runtime_error(std::string(__func__) + ": keypool entry misclassified");
            }

    -   如果密钥池中保存的公钥是无效的，那么抛出异常。

            if (!keypool.vchPubKey.IsValid()) {
                throw std::runtime_error(std::string(__func__) + ": keypool entry invalid");
            }

    -   从 `m_pool_key_to_index` 集合中消除对应的索引。

            m_pool_key_to_index.erase(keypool.vchPubKey.GetID());

    -   返回真。

3.  调用 `KeepKey`

4.  从密钥池中取出对应的公钥。


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

