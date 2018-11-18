#   SigningProvider

一个支持签名的接口，keystores 实现了它的接口。

#   CKeyStore

一个纯虚函数的基类。

#   CBasicKeyStore

基本的 KeyStore，继承自 `CKeyStore`，把密钥保存在 `address->secret` 映射中。

mapKeys，保存私钥的映射。

mapWatchKeys，保存公钥的映射。

mapScripts，脚本的映射。

setWatchOnly，脚本的向量。


#   CCryptoKeyStore

保存加密私钥的 Keystore，继承自基本的　key store。

fUseCrypto 一个标志，如果为真，则 mapKeys 为空，如果为假，则 vMasterKey 为空。在构造函数中被设置为假。

