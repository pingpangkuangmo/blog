# 1 Hive系列目录

-	[Metastore认证和验权]()
-	[HiveServer2认证和验权]()
-	[sentry的权限设计]()
-	[HiveServer2接入sentry后的认证和验权]()
-	[Metastore接入sentry后的认证和验权]()
-	[HiveServer2 thrift服务详细分析]()

# 2 Metastore

## 2.1 Metastore服务介绍

metastore主要维护2种数据：

-	数据库、表、分区等数据，算是DDL操作
-	权限、角色类的数据，算是DCL操作

metastore通过开启thrift rpc服务，开启对上述2种数据的操作接口，接口即 ThriftHiveMetastore.Iface 内容见下图

第一种数据的操作如下：

![数据库、表等操作](https://static.oschina.net/uploads/img/201606/18182802_b1rR.png "数据库、表等操作")

第二种数据的操作如下：

![角色权限等操作](https://static.oschina.net/uploads/img/201606/18183020_UtSQ.png "角色权限等操作")

metastore对该接口的实现为：HMSHandler，它主要有以下属性：

-	ThreadLocal<RawStore\> threadLocalMS
	
	RawStore主要用于和数据库打交道，存储上述2种相关数据，RawStore和当前线程进行绑定，默认实现是ObjectStore

-	List<MetaStorePreEventListener\> preListeners
	
	当上述2种数据将要发生变化的时候，都会首先调用上述MetaStorePreEventListener执行一些预处理操作，如验证一个用户是否有权限来执行该操作

-	List<MetaStoreEventListener\> listeners

	当上述第一种数据发生变化后，会调用上述MetaStoreEventListener执行一些处理

## 2.2 Metastore服务的认证和验权

由上述可以了解到，在metastore的数据发生修改之前可以进行验权操作，hive默认提供了一个AuthorizationPreEventListener实现了上述MetaStorePreEventListener接口执行一些验权操作

![metastore的验权处](https://static.oschina.net/uploads/img/201606/18185253_Q72B.png "metastore的验权处")

从上面可以看到，仅仅对DDL操作进行了验权，并没有对DCL操作进行验权，这也是一个比较奇怪的地方。即如果你能直连到metastore服务，就可以随意的进行授权操作。

接下来我们详细看看这个认证和授权的过程

涉及到2个接口：

-	HiveMetastoreAuthenticationProvider：认证提供者，用于用户的认证，一个HiveMetastoreAuthenticationProvider对象对应一个用户，该对象包含用户的userName和groups信息

-	HiveMetastoreAuthorizationProvider：授权提供者，用于用户的验权

下面分别来详细说明

## 2.2.1  Metastore服务的认证

上述HiveMetastoreAuthenticationProvider的默认实现是HadoopDefaultMetastoreAuthenticator，通过hadoop中的UserGroupInformation来获取当前用户名和组的信息，如下所示

![获取用户信息](https://static.oschina.net/uploads/img/201606/18191340_RumK.png "获取用户信息")

metastore的client端用户是如何被传递到这里的呢？

这就涉及到了metastore开启的thrift rpc服务了，metastore使用的是thrift的TThreadPoolServer来作为server。这种模式下，会启动一个线程池，每来一个客户端连接就取出一个线程来专门处理该连接，该连接就一直占用该线程了，这种方式就是传统的BIO方式，能够支持的并发量比较小。

metastore开启的rpc服务分成3种类型：

-	使用sasl方式：
	是一种安全的方式，使用kerberos认证用户的身份，这一部分内容比较多，可能之后再详细分析这一块。

-	setUgi方式：

	是一种非安全的方式，即并没有对用户的身份进行合法性验证。当hive.metastore.execute.setugi=true时即采用这种模式。这种模式下如下操作：
	
	-	client端会通过set_ugi方法来向服务器端传递用户的ugi信息

	-	服务器端将用户的ugi信息绑定到当前连接上

	-	client端向服务器端发送操作数据的请求，服务器端取出连接中的ugi信息，使用ugi的doas方法来执行操作

	-	client占用服务器端的一个线程，会创建出一个HiveMetastoreAuthenticationProvider对象，该对象在创建过程中要获取当前的ugi信息，即获取到上述ugi信息，并且把client的上述HiveMetastoreAuthenticationProvider对象绑定在该线程中，至此便可以得到client的用户信息。而groups信息则是通过hadoop默认的方式即获取本metastore服务所在的机器上，该用户所属的groups信息。

-	其他：

	就仅仅是将client的ip绑定到当前线程上，就没有所谓的用户信息了

上述的sasl也是通过ugi.doas方式来供后续获取用户信息的，但是sasl多了用户身份合法性的验证过程。而setUgi方式client端传过来什么用户就认为是什么用户，所以只有sasl方式才是安全的方式，其他2种是非安全方式。

## 2.2.2  Metastore服务的验权

上一部分我们获取到了用户信息，下面就是要验证用户是否有权限执行相关操作，验权接口就是HiveMetastoreAuthorizationProvider，实现类如下：

![metastore验权实现类](https://static.oschina.net/uploads/img/201606/18225248_vOTy.png "metastore验权实现类")

默认是DefaultHiveMetastoreAuthorizationProvider。

-	DefaultHiveMetastoreAuthorizationProvider：

	根据userName和groups信息从数据库中获取权限信息来验证用户是否有权限

-	StorageBasedAuthorizationProvider：

	判断一个用户是否对数据库、表等是否有权限依据该用户在HDFS上对这些文件是否有对应的权限

# 3 总结

总结如下：

-	metastore只做了DDL操作的相关认证，并没有做DCL操作的认证，一旦通过hive cli连接metastore，用户就可以随意进行赋权操作

-	根据用户和用户所属的groups信息来获取权限，大数据系统中用户和groups关系的维护最好是自己统一维护，而不是借助默认的linux机器上用户和groups的关系，不然之后会有很多麻烦的。


画了一个默认情况下简单的示意图如下：

![输入图片说明](https://static.oschina.net/uploads/img/201606/18233127_jFSr.png "在这里输入图片标题")


# 4 结束语

本篇简单介绍了metastore端的认证和授权，下一篇就再介绍下HiveServer2的认证和授权