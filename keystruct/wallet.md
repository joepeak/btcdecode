#	钱包及地址相关结构

##  钱包

**钱包是 keystore 的扩展，管理着交易和余额，提供创建交易的能力，同时还实现了 `CValidationInterface` 的某些接口**。

具有下列重要属性：

-   fAbortRescan

    是否终止扫描

-   fScanningWallet

    是否正在扫描

-   mutexScanning

-   WalletRescanReserver

-   encrypted_batch

-   nWalletVersion

-   nWalletMaxVersion

-   nNextResend

-   nLastResend

-   fBroadcastTransactions

    是否广播交易。创建钱包时，根据启动参数 `-walletbroadcast` 设置，默认为真。
    
-   TxSpends

-   mapTxSpends

-   hdChain

    确定分层链数据模型，类型为 `CHDChain`

-   setInternalKeyPool

-   setExternalKeyPool

-   set_pre_split_keypool

-   m_max_keypool_index

-   m_pool_key_to_index

-   m_wallet_flags

-   nTimeFirstKey

-   m_name

-   database

-   m_last_block_processed

-   mapKeyMetadata

-   m_script_metadata

-   MasterKeyMap

-   mapMasterKeys

-   nMasterKeyMaxID

-   mapWallet

-   TxItems

-   wtxOrdered

-   nOrderPosNext

-   nAccountingEntryNumber

-   mapAddressBook

-   setLockedCoins


##	钱包功能

系统中钱包功能由枚举类 `WalletFeature` 表示，钱包具有下面这些功能：

-	FEATURE_BASE

	最早钱包支持的功能，仅用于 `getwalletinfo` 输出。

-	FEATURE_WALLETCRYPT

	钱包支持加密

-	FEATURE_COMPRPUBKEY

	钱包支持压缩公钥

-	FEATURE_HD

	BIP32 之后的分层密钥推导 HD 钱包

-	FEATURE_HD_SPLIT

	具有 HD 链分割的钱包

-	FEATURE_NO_DEFAULT_KEY

	Wallet without a default key written

-	FEATURE_PRE_SPLIT_KEYPOOL

	Upgraded to HD SPLIT and can have a pre-split keypool

-	FEATURE_LATEST

	等于 `FEATURE_PRE_SPLIT_KEYPOOL`

##	输出类型

输出类型由枚举类 `OutputType` 表示，具体类型有：

-	LEGACY

-	P2SH_SEGWIT

	默认的地址类型

-	BECH32

-	CHANGE_AUTO

##	CMerkleTx

A transaction with a merkle branch linking it to the block chain.

具有下列重要的属性：

-	ABANDON_HASH

-	tx

	交易对象，类型为 `CTransactionRef`

-	hashBlock

-	nIndex


##	CWalletTx

继承自 `CMerkleTx`。

具有下列重要的属性：

-	pwallet

	指向钱包的指针，类型为 `CWallet`

-	mapValue

-	vOrderForm

-	fTimeReceivedIsTxTime

-	nTimeReceived

-	nTimeSmart

-	fFromMe

-	nOrderPos

	交易列表中的位置

-	m_it_wtxOrdered

-	fDebitCached

-	fCreditCached

-	fImmatureCreditCached

-	fAvailableCreditCached

-	fWatchDebitCached

-	fWatchCreditCached

-	fImmatureWatchCreditCached

-	fAvailableWatchCreditCached

-	fChangeCached

-	fInMempool

-	nDebitCached

-	nCreditCached

-	nImmatureCreditCached

-	nAvailableCreditCached

-	nWatchDebitCached

-	nWatchCreditCached

-	nImmatureWatchCreditCached

-	nAvailableWatchCreditCached

-	nChangeCached


##	交易输出

交易输出由类 `COutput` 表示，具有下列重要属性：

-	tx

	指向钱包的交易，类型为 `CWalletTx`

-	i

-	nDepth

-	nInputBytes

-	fSpendable

	是否有私钥可以花费这笔输出

-	fSolvable

	是否我们知道如何花费这笔输出

-	use_max_sig

-	fSafe

	这笔输出是否可以安全地花费。


##	钱包私钥

包含失效日期的钱包私钥，类型为 `CWalletKey`，具有下列重要属性：

-	vchPrivKey

	私钥，类型为 `CPrivKey`

-	nTimeCreated

-	nTimeExpires

-	strComment


##	CReserveKey

继承自 `CReserveScript`

-	pwallet

-	nIndex

-	vchPubKey

-	fInternal


##	WalletRescanReserver

-	m_wallet

-	m_could_reserve



-	
