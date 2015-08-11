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

我们知道node的类型分为四种类型：

-	PERSISTENT：持久型节点
-	PERSISTENT_SEQUENTIAL：持久型顺序型节点
-	EPHEMERAL：临时型节点
-	EPHEMERAL_SEQUENTIAL：临时型顺序型节点

前两者持久型节点和后两者临时型节点的不同之处就在于，一旦当客户端session过期，则会清除临时型节点，不会清除持久型节点，除非去执行删除操作。

而顺序型节点，则是每次创建一个节点，会在一个节点路径的后面加上父节点的cversion版本号（即该父节点的所有子节点一旦发生变化，就会自增该版本号）的格式化形式，如下：

![顺序型节点来来历](https://static.oschina.net/uploads/img/201508/06071819_oZE9.png "顺序型节点来来历")

可以看到是将父节点的cversion版本号以10进制形式输出，宽度是10位，不足的话前面补0。所以是在执行DataTree创建node方法之前就已经定好了path路径的。

再来看下是如何区分持久型和临时型节点的呢？

在DataTree创建node方法会传递一个ephemeralOwner参数，当客户端选择的是持久型节点，给出的sessionId为0，当为临时型节点时，给出客户端的sessionId,如下：

![输入图片说明](https://static.oschina.net/uploads/img/201508/06072814_QcGi.png "在这里输入图片标题")

先来看下DataTree创建node方法的方法：

![输入图片说明](https://static.oschina.net/uploads/img/201508/06073140_syp2.png "在这里输入图片标题")

-	先判断父节点存不存在，不存在的话，报错。
-	然后检查父节点的所有子节点是否已存在要创建的节点，如果存在报错。
-	创建出节点，并存放到DataTree的ConcurrentHashMap<String, DataNode> nodes属性中，见上文描述
-	判断该节点是否是临时节点，如果是临时节点，则ephemeralOwner参数即为客户端的sessionId。然后以sessionId为key，存储该客户端所创建的所有临时节点到DataTree的Map<Long, HashSet<String>> ephemerals属性中，见上文描述

ZKDatabase先暂时介绍到这里，之后抽出一篇文章单独介绍DataTree和FileTxnSnapLog。

##3.3 ZooKeeperServer请求处理器链介绍
ZooKeeper使用请求处理器链的方式来处理请求，先看下请求处理器的定义RequestProcessor:

![RequestProcessor定义](https://static.oschina.net/uploads/img/201508/07072024_h4Gr.png "RequestProcessor定义")

从注释上可以看到几个要点：

-	RequestProcessor是以责任链的形式来处理事务的。
-	请求是被顺序的进行处理的，单机版、集群版的Leader、Follower略有不同
-	对于请求的处理，是通过processRequest(Request request)方法来处理的。有些处理器是一个线程，即请求被扔到该线程中进行处理
-	当调用shutdown时，也会关闭它所关联的RequestProcessor

来看下ZooKeeperServer请求处理器链的具体情况：

![ZooKeeperServer请求处理器链](https://static.oschina.net/uploads/img/201508/07075617_AElm.png "ZooKeeperServer请求处理器链")

即PrepRequestProcessor-》SyncRequestProcessor-》FinalRequestProcessor

来一个一个具体看看：

###3.3.1 PrepRequestProcessor处理器

主要内容：对请求进行区分是否是事务请求，如果是事务请求则创建出事务请求头，同时执行一些检查操作。

大体属性如下：

![PrepRequestProcessor属性](https://static.oschina.net/uploads/img/201508/08113256_o8j9.png "PrepRequestProcessor属性")

-	LinkedBlockingQueue<Request> submittedRequests：提交的用户请求
-	RequestProcessor nextProcessor：下一个请求处理器
-	ZooKeeperServer zks：服务器对象

PrepRequestProcessor所实现的processRequest接口方法即为：将该请求放入submittedRequests请求队列中。同时PrepRequestProcessor又是一个线程，在run方法中又会不断的取出上述用户提交的请求，进行处理，整个处理过程如下：

![创建事务请求头](https://static.oschina.net/uploads/img/201508/08114347_XpgI.png "创建事务请求头")

对于增删改等影响数据状态的操作都被认为是事务，需要创建出事务请求头。

![只需验证session](https://static.oschina.net/uploads/img/201508/08114632_QYRC.png "只需验证session")

createSession、closeSession也属于事务操作，而那些获取数据的操作则不属于事务操作，只需要验证下sessionId是否合法等

处理完成之后就交给了下一个处理器继续处理该请求。

我们以创建session和创建节点为例，来具体看下代码：

先看下创建session，即如下代码：

![session创建处理](https://static.oschina.net/uploads/img/201508/08181502_bkJg.png "session创建处理")

首先会为该request获取一个事务id即zxid，该zxid的值来自于ZooKeeper服务器的一个hzxid变量，默认是0，每来一个请求就会执行自增操作。

![创建事务请求头](https://static.oschina.net/uploads/img/201508/08182446_0gDL.png "创建事务请求头")
	
![创建session的具体内容](https://static.oschina.net/uploads/img/201508/08182740_IlXi.png "创建session的具体内容")

首先获取客户端传递过来的sessionTimeout时间，然后使用ZooKeeperServer的sessionTracker来创建一个session，同时为该session的owner属性赋值，但是对于创建session的request请求，并没有为owner赋值。而是在创建其他请求的时候才会为请求的owner赋值为本机器。

代码见证如下：

创建session的request如下：

![输入图片说明](https://static.oschina.net/uploads/img/201508/08184808_Usqm.png "在这里输入图片标题")

创建其他的request的如下：

![输入图片说明](https://static.oschina.net/uploads/img/201508/08184718_Lsmw.png "在这里输入图片标题")

接下来看看创建一个node的处理：

-	首先进行的是session检查。

	![session及owner的检查](https://static.oschina.net/uploads/img/201508/08185231_cYhk.png "session及owner的检查")

	先检查服务器端该sessionId还是否存在，如果不存在则表示已经过期，抛出SessionExpiredException异常。如果session存在，owner为空，则会对owner进行赋值。如果owner存在则进行owner核对，如果不一致抛出SessionMovedException异常。

	第一次创建session后，该session的owner是为空的，之后的请求操作owner都是有值的，此时则会为该session赋值。

	我们想象下这样的场景：客户端连接一台服务器server1，该客户端拥有的session的owner是server1，客户端发送操作请求，由于网络原因造成请求阻塞，客户端认为server1不稳定，则会拿着刚才的session去连接另一台服务器server2，连接成功后，该session对应的owner被设置为null了（根据上文知道创建session的时候，owner会被清空），然后继续在server2上执行同样的操作，此时会为该session的owner属性赋值为server2，如果之前对server1的请求此时终于到达server1了，此request的owner是server1，则在检查的时候，发现该owner不一致，server1则会抛出SessionMovedException异常。即session的owner已经变化了的异常。则会阻止该请求的执行，防止了重复执行相同的操作。这里再留个疑问：为什么当创建session的时候要清空owner呢？

-	反序列化出CreateRequest对象，获取要创建的路径，同时获取父路径，检查该父路径是否存在，如果不存在抛出异常NoNodeException。如果存在获取父路径的修改记录，验证对父路径是否有修改权限

-	从CreateRequest中获取用户创建的node的类型，如果是临时节点的话，则根据父路径的子节点的版本cversion，来生成该临时节点的路径后缀部分。然后验证该路径是否存在，如果存在则抛出NodeExistsException异常

-	判断父节点是否是临时节点，如果是临时节点则不应该有子节点，抛出NoChildrenForEphemeralsException异常，我认为这部分代码该判断应该是提前应该做的，而不是留到现在才来判断

-	如果该节点是临时节点，则为该节点ephemeralOwner属性设置为对应的sessionId，如果是永久节点则设置为0。而DataTree则是依据ephemeralOwner是否为0，来判断是否是临时节点还是持久节点，如果是临时节点，则会另外存储一份数据，以sessionId为key，即列出了每个sessionId所包含的所有临时节点，一旦该sessionId失效，则直接拿出该列表进行清除操作即可，不用再去遍历所有的节点了。

-	产生两条变化记录，分别是父节点的子节点列表变化的记录，和要创建的节点的创建记录。

至此便完成预处理操作。该交给下一个RequestProcessor处理器来处理了

###3.3.2 SyncRequestProcessor处理器

主要对事务请求进行日志记录，同时事务请求达到一定次数后，就会执行一次快照。

主要属性如下：

![SyncRequestProcessor属性](https://static.oschina.net/uploads/img/201508/10075512_zXMD.png "SyncRequestProcessor属性")

-	ZooKeeperServer zks：ZooKeeper服务器对象
-	LinkedBlockingQueue<Request> queuedRequests：提交的请求（包括事务请求和非事务请求）
-	RequestProcessor nextProcessor：下一个请求处理器
-	Thread snapInProcess：执行一次快照任务的线程
-	LinkedList<Request> toFlush：那些已经被记录到日志文件中但还未被flush到磁盘上的事务请求
-	int snapCount：发生了snapCount次的事务日志记录，就会执行一次快照
-	int randRoll：上述是一个对所有服务器都统一的配置数据，为了避免所有的服务器在同一时刻执行快照任务，实际情况为发生了（snapCount / 2 + randRoll）次的事务日志记录，就会执行一次快照。randRoll的计算方式如下：

		r.nextInt(snapCount/2)

接下来就详细看下SyncRequestProcessor（也是一个线程）的详细实现：

对于RequestProcessor定义的接口：processRequest(Request request)，SyncRequestProcessor和PrepRequestProcessor一样，都是讲请求放入阻塞式队列中，然后在线程run方法中执行相应的逻辑操作。

首先还是从LinkedBlockingQueue<Request> queuedRequests队列中取出一个Request，处理如下：

![SyncRequestProcessor处理过程](https://static.oschina.net/uploads/img/201508/10081710_QtdC.png "SyncRequestProcessor处理过程")

-	第一步：将该请求添加到事务日志中，这一部分会区分Request是事务请求还是非事务请求，依据就是前一个处理器PrepRequestProcessor为Request加上的事务请求头。如果是事务请求，则添加成功后返回true，添加成功即为将该请求序列化到一个指定的文件中。如果是非事务请求，直接返回false。
-	第二步：如果是事务请求，添加到事务日志中后，logCount++，该logCount就是用于记录已经执行多少次事务请求序列化到日志中了。
-	第三步：一旦logCount超过（snapCount / 2 + randRoll）次后，就需要执行一次快照了。
-	第四步：先将当前的事务日志记录flush到磁盘中，然后设置当前流为null，以便下一次事务日志记录重新开启一个新的文件来记录
-	第五步：创建一个ZooKeeperThread线程，用于执行一次快照任务，则会把当前的dataTree和sessionsWithTimeouts信息序列化到一个文件中。
-	第六七步：如果是非事务请求的话，则会直接交给下一个RequestProcessor处理器来处理。我们看到这里还加上了一个toFlush.isEmpty()的判断，即之前没有请求遗留，只有在这样的条件下才会直接交给下一个RequestProcessor处理器来处理，主要是为了保证请求的顺序性。如果之前还有遗留的请求，则后来的请求不能被先处理。

上述的请求除了直接被下一个处理器处理的情况，其余大部分都会被保存到LinkedList<Request> toFlush中，什么时候才会被执行呢？

![toFlush中的request被执行1](https://static.oschina.net/uploads/img/201508/10083715_WfJr.png "toFlush中的request被执行1")

![toFlush中的request被执行2](https://static.oschina.net/uploads/img/201508/10083918_j9z7.png "toFlush中的request被执行2")

两种情况下会被执行flush：

-	当request数量超过1000
-	当没有请求到来的时候

来看下具体的flush过程：

![flush过程](https://static.oschina.net/uploads/img/201508/10084347_zN1q.png "flush过程")

-	第一步：执行事务日志文件执行commit操作。上述rollLog操作仅仅是先flush，然后设置当前日志记录流为null，以便下一次重新开启一个新的事务日志文件，同时这些流都会被保存到LinkedList<FileOutputStream> streamsToFlush属性中，commit操作则是先flush这些所有的流，然后执行这些流的close操作。
-	第二步：便是将请求交给下一个处理器来处理

至此SyncRequestProcessor的内容也完成了，接下来就是下一个请求处理器即FinalRequestProcessor

###3.3.2 FinalRequestProcessor处理器

作为处理器链上的最后一个处理器，负责执行请求的具体任务，前面几个处理器都是辅助操作，如PrepRequestProcessor为请求添加事务请求头和执行一些检查工作，SyncRequestProcessor也仅仅是把该请求记录下来保存到事务日志中。该请求的具体内容，如获取所有的子节点，创建node的这些具体的操作就是由FinalRequestProcessor来完成的。

下面就来详细看看FinalRequestProcessor处理request的过程

![FinalRequestProcessor的处理内容](https://static.oschina.net/uploads/img/201508/11080051_wAyR.png "FinalRequestProcessor的处理内容")

-	对于request是顺序执行，所以要删除那些zxid小于当前request的zxid的outstandingChanges、以及outstandingChangesForPath
。这里就有一个疑问：outstandingChanges数据是由PrepRequestProcessor在预处理事务请求头的时候产生的，他们又被谁来消费呢？他们主要作用是什么？

##3.4 ServerStats介绍

