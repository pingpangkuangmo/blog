#1 系列目录

-	[ZooKeeper源码研究系列（1）源码环境搭建](http://my.oschina.net/pingpangkuangmo/blog/484955)
-	[ZooKeeper源码研究系列（2）客户端创建连接过程分析](http://my.oschina.net/pingpangkuangmo/blog/486780)

#2 单机版服务器启动方式

单机版的服务器启动，使用ZooKeeperServerMain的main函数来启动，参数分为两种：

-	只有一个参数：表示为一个配置文件地址
-	有2~4个参数：分别表示端口、dataDir、tickTime、maxClientCnxns

详细介绍见开篇的介绍[运行ZooKeeper](http://my.oschina.net/pingpangkuangmo/blog/484955#OSC_h1_7)

接下来看下启动的整个过程：

![输入图片说明](https://static.oschina.net/uploads/img/201508/04071117_YNcA.png "在这里输入图片标题")

-	第一步：创建一个ZooKeeperServer，代表着一个服务器对象
-	第二步：根据配置参数dataLogDir和dataDir创建出用于管理事务日志和快照的对象FileTxnSnapLog
-	第三步：对ZooKeeperServer设置一些配置参数，如tickTime、minSessionTimeout、maxSessionTimeout
-	第四步：创建ServerCnxnFactory，用于创建ServerSocket，等待客户端的socket连接
-	第五步：启动ZooKeeperServer服务

下面分别来详细说明

#3 创建一个ZooKeeperServer服务器对象

ZooKeeperServer是单机版才使用的服务器对象，集群版都是使用的是它的子类，来看下继承类图

![ZooKeeperServer类图](https://static.oschina.net/uploads/img/201508/04072333_3MBV.png "ZooKeeperServer类图")

可以看到，集群版分别用的是LeaderZooKeeperServer、FollowerZooKeeperServer、ObserverZooKeeperServer。后两者都属于LearnerZooKeeperServer。

##3.1 ZooKeeperServer的重要属性

![ZooKeeperServer的重要属性](https://static.oschina.net/uploads/img/201508/04072819_rtqC.png "ZooKeeperServer的重要属性")

-	tickTime:默认3000ms，用于计算默认的minSessionTimeout、maxSessionTimeout。计算方式如下：

		public int getMinSessionTimeout() {
        	return minSessionTimeout == -1 ? tickTime * 2 : minSessionTimeout;
    	}

		public int getMaxSessionTimeout() {
        	return maxSessionTimeout == -1 ? tickTime * 20 : maxSessionTimeout;
    	}

	同时还用于指定SessionTrackerImpl的执行过期检查的周期时间，详细见说明[使用sessionTracker的session过期检查](http://my.oschina.net/pingpangkuangmo/blog/486780#OSC_h2_8)

-	minSessionTimeout、maxSessionTimeout：用于限制客户段给出的sessionTimeout时间
-	SessionTracker sessionTracker：负责创建和管理session，同时负责定时进行过期检查
-	ZKDatabase zkDb：用于存储ZooKeeper树形数据的模型
-	FileTxnSnapLog txnLogFactory：负责管理事务日志和快照日志文件，能根据它加载出数据到ZKDatabase中，同时能将ZKDatabase中的数据以及session保存到快照日志文件中。后面会详细说明FileTxnSnapLog。

	操作如下：
		
		new ZKDatabase(txnLogFactory)

		txnLogFactory.save(zkDb.getDataTree(), zkDb.getSessionWithTimeOuts());

-	RequestProcessor firstProcessor：ZooKeeperServer请求处理器链中的第一个处理器

-	long hzxid：ZooKeeperServer最大的事务编号，每来一个事务请求，都会分配一个事务编号

-	ServerCnxnFactory serverCnxnFactory：负责创建ServerSocket，接受客户端的socket连接
-	ServerStats serverStats：负责统计server的运行状态

##3.2 ZKDatabase介绍

先来看下ZKDatabase的注释和属性

![ZKDatabase的注释和属性](https://static.oschina.net/uploads/img/201508/04190119_Vf9w.png "ZKDatabase的注释和属性")

从注释中可以看到ZKDatabase中所包含的信息有：

-	sessions信息，即ConcurrentHashMap<Long, Integer> sessionsWithTimeouts，也就是说仅仅会保存sessionId对应的timeout时间
-	DataTree：即ZooKeeper的内存节点信息
-	LinkedList<Proposal> committedLog：用于保存最近提交的一些事物

来重点看下DataTree的实现：

-	ConcurrentHashMap<String, DataNode> nodes =new ConcurrentHashMap<String, DataNode>();

	维护了path对应的DataNode。每个DataNode内容如下：
	
	![DataNode内容](https://static.oschina.net/uploads/img/201508/04195703_gKTY.png "DataNode内容")

	有DataNode parent和Set<String> children，同时byte data[]存储本节点的数据。StatPersisted stat存储本节点的状态信息

-	Map<Long, HashSet<String>> ephemerals =new ConcurrentHashMap<Long, HashSet<String>>()

	维护了每个session对应的临时节点的集合

-	WatchManager dataWatches、WatchManager childWatches分别用于管理节点自身数据更新的事件触发和该节点的所有子节点变动的事件触发。

	每个WatchManager的结构如下：

	![WatchManager的结构](https://static.oschina.net/uploads/img/201508/04200955_u6qy.png "WatchManager的结构")

	watchTable维护着每个path对应的Watcher。watch2Paths维护着每个Watcher监控的所有path，即每个Watcher是可以监控多个path的。在服务器端Watcher的实现其实是ServerCnxn，如下：

		public abstract class ServerCnxn implements Stats, Watcher
	
	而每个ServerCnxn则代表服务器端为每个客户端分配的handler，负责与客户端进行通信。客户端每次对某个path注册的Watcher,在传输给服务器端的时候仅仅是传输一个boolean值，即是否监听某个path，并没有把我们自定义注册的Watcher传输到服务器端（况且Watcher也不能序列化），而是在本地客户端进行存储，存储着对某个path注册的Watcher。服务器端接收到该boolean值之后，如果为true，则把该客户端对应的ServerCnxn作为Watcher存储到上述WatchManager中，即上述WatchManager中存储的是一个个ServerCnxn实例。一旦服务器端数据变化，触发对应的ServerCnxn，ServerCnxn然后把该事件又传递客户端，客户端这时才会真正引发我们自定义注册的Watcher。

	上面只是简单描述了一下，之后的文章会详细源码分析整个过程。

DataTree就负责进行node的增删改查。

要解决的问题：

如何创建一个node，根据node的类型：持久型节点、持久顺序型节点、临时节点、临时顺序型节点

通过 stat.setEphemeralOwner(ephemeralOwner);中的ephemeralOwner是否为0来判断是否是持久型和临时节点

在调用DataTree的createNode方法时已经是变更过的path路径了。

顺序型节点如何实现呢？




##3.3 ZooKeeperServer请求处理器链介绍

##3.4 ServerStats介绍

