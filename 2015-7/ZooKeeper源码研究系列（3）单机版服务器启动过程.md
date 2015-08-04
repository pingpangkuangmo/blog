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

##3.3 ZooKeeperServer请求处理器链介绍

##3.4 ServerStats介绍

