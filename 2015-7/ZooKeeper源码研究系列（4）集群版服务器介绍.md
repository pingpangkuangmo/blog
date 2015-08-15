#1 系列目录

-	[ZooKeeper源码研究系列（1）源码环境搭建](http://my.oschina.net/pingpangkuangmo/blog/484955)
-	[ZooKeeper源码研究系列（2）客户端创建连接过程分析](http://my.oschina.net/pingpangkuangmo/blog/486780)

#2 集群版服务器启动过程

启动类是org.apache.zookeeper.server.quorum.QuorumPeerMain，启动参数就是配置文件的地址

##2.1 配置文件说明

来看下一个简单的配置文件内容：

	tickTime=4000
	initLimit=10
	syncLimit=5
	dataDir=D:\\zk-test\\datadir\\server1
	clientPort=2181
	maxClientCnxns=60
	
	server.1=localhost:2881:3881
	server.2=localhost:2882:3882
	server.3=localhost:2883:3883

-	tickTime值，单位ms,默认3000

	-	用途1：用于指定session检查的间隔

		服务器会每隔一段时间检查一次连接它的客户端的session是否过期。该间隔就是tickTime。

	-	用途2：用于给出默认的minSessionTimeout和maxSessionTimeout

		如果没有给出maxSessionTimeout和minSessionTimeout（为-1），则minSessionTimeout和maxSessionTimeout的取值如下：

			minSessionTimeout == -1 ? tickTime * 2 : minSessionTimeout;
			maxSessionTimeout == -1 ? tickTime * 20 : maxSessionTimeout;
		
		分别是tickTime的2倍和20倍。

		客户端代码在创建ZooKeeper对象的时候会给出一个sessionTimeout时间，而上述的minSessionTimeout和maxSessionTimeout就是用来约束客户端的sessionTimeout

-	initLimit：后面源码详细说明

-	syncLimit：后面源码详细说明

-	dataDir:用于存储数据快照的目录

-	dataLogDir：用于存储事务日志的目录，如果没有指定，则和dataDir保持一致

-	clientPort：对客户端暴漏的连接端口

-	maxClientCnxns值，用于指定服务器端最大的连接数。

-	集群的server配置，格式为server.A=B:C:D

	A:即为集群中server的id标示，很多地方用到它，如选举过程中，就是通过id来识别是哪台服务器的投票。如初始化sessionId的时候，也用到id来防止sessionId出现重复。

	B：即该服务器的host地址或者ip地址。

	C：一旦Leader选举成功后，非Leader服务器都要和Leader服务器建立tcp连接进行通信。该通信端口即为C

	D：在Leader选举阶段，每个服务器之间相互连接（上述serverId大的会主动连接serverId小的server），进行投票选举的事宜，D即为投票选举时的通信端口
	
	上述配置是每台服务器都要知道的集群配置，同时要求在dataDir目录中创建一个myid文件，里面写上上述serverid中的一个id值，即表明某台服务器所属的编号

##2.2 集群版服务器启动概述

![输入图片说明](https://static.oschina.net/uploads/img/201508/15183729_iL6W.png "在这里输入图片标题")

#3 集群版建立连接过程