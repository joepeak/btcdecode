#   比特币关键数据结构之地址

##	1、CNetAddr 类

这个类的定义在 `netaddress.h` 文件中，表示节点的 IPV4 或 IPV6 地址。

主要属性如下：

-	ip

	IP地址，无符号16位的字符串。

-	scopeId


##	2、CService

这个类的定义同样在 `netaddress.h` 文件中，组合了 IP 地址和 TCP 的端口号，继承于 `CNetAddr` 类。

主要属性如下：

-	port

	主机的端口

##	3、CAddress

这个类的定义在 `protocol.h` 文件中，表示对等节点信息，继承于 `CService` 类。

主要属性如下：

-	nServices

	节点所支持的服务。

-	nTime


##	4、CAddrInfo

这个类的定义在 `addrman.h` 文件中，继承于 `CAddress`，增加了一些统计信息。

主要属性如下：

-	nLastTry

-	nLastCountAttempt

-	source

-	nLastSuccess

-	nAttempts

-	nRefCount

-	fInTried

-	nRandomPos


##	5、CAddrMan

这个类的定义在 `addrman.h` 文件中，表示随机的地址管理器。

主要属性如下：

-	nIdCount

	最后一个使用的 nId，也是下面属性的 Key。

-	mapInfo

	地址信息的映射，Key 是 nIdCount，Value 为 `CAddrInfo` 对象。在获得地址成功后，保存在本属性中。


