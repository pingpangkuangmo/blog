#1 系列目录

-	[ZooKeeper源码研究系列（1）源码环境搭建](http://my.oschina.net/pingpangkuangmo/blog/484955)
-	[ZooKeeper源码研究系列（2）客户端创建连接过程分析](http://my.oschina.net/pingpangkuangmo/blog/486780)
-	[ZooKeeper源码研究系列（3）单机版服务器介绍](http://my.oschina.net/pingpangkuangmo/blog/491673)

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
	
	-	用途3：作为initLimit和syncLimit时间的基数，见下面

-	initLimit：在初始化阶段和Leader的通信的读取超时时间，即当调用socket的InputStream的read方法时最大阻塞时间不能超过initLimit*tickTime。设置如下：

	![initLimit读取超时时间](https://static.oschina.net/uploads/img/201508/17081952_dsWr.png "initLimit读取超时时间")

	initLimit还会作为初始化阶段收集相关响应的总时间，一旦超过该时间，还没有过半的机器进行响应，则抛出InterruptedException的timeout异常

-	syncLimit：在初始化阶段之后的请求阶段和Leader通信的读取超时时间，即对Leader的一次请求到响应的总时间不能超过syncLimit*tickTime时间。Follower和Leader之间的socket的超时时间初始化阶段是前者，当初始化完毕又设置到后者时间上。设置如下：

	![syncLimit读取超时时间](https://static.oschina.net/uploads/img/201508/17082136_N8HO.png "syncLimit读取超时时间")

	syncLimit还会作为与Leader的连接超时时间，如下：

	![与Leader的连接超时时间](https://static.oschina.net/uploads/img/201508/17082447_RK1x.png "与Leader的连接超时时间")

	

-	dataDir:用于存储数据快照的目录

-	dataLogDir：用于存储事务日志的目录，如果没有指定，则和dataDir保持一致

-	clientPort：对客户端暴漏的连接端口

-	maxClientCnxns值，用于指定服务器端最大的连接数。

-	集群的server配置，一种格式为server.A=B:C:D，还有其他格式，具体可以去看QuorumPeerConfig源码解析这一块

	A:即为集群中server的id标示，很多地方用到它，如选举过程中，就是通过id来识别是哪台服务器的投票。如初始化sessionId的时候，也用到id来防止sessionId出现重复。

	B：即该服务器的host地址或者ip地址。

	C：一旦Leader选举成功后，非Leader服务器都要和Leader服务器建立tcp连接进行通信。该通信端口即为C

	D：在Leader选举阶段，每个服务器之间相互连接（上述serverId大的会主动连接serverId小的server），进行投票选举的事宜，D即为投票选举时的通信端口
	
	上述配置是每台服务器都要知道的集群配置，同时要求在dataDir目录中创建一个myid文件，里面写上上述serverid中的一个id值，即表明某台服务器所属的编号

##2.2 集群版服务器启动概述

我们由前一篇文章知道了，单机版的服务器启动，就是创建了一个ZooKeeperServer对象。我们需要再次熟悉下ZooKeeperServer的类图，如下：

![输入图片说明](https://static.oschina.net/uploads/img/201508/16204328_r3DX.png "在这里输入图片标题")

可见Leader服务器要使用LeaderZooKeeperServer，Follower服务器要使用FollowerZooKeeperServer。而集群版服务器启动后，可能是Leader或者Follower。在运行过程中角色还会进行自动更换，即自动更换使用不同的ZooKeeperServer子类。此时就需要一个代理对象，用于角色的更换、所使用的ZooKeeperServer的子类的更换。这就是QuorumPeer，如下图

![输入图片说明](https://static.oschina.net/uploads/img/201508/15183729_iL6W.png "在这里输入图片标题")

这里面很多的配置属性都交给了QuorumPeer，由它传递给底层所使用的ZooKeeperServer子类。

来详细看看这些配置属性：

-	ServerCnxnFactory cnxnFactory：负责和客户端建立连接和通信

-	FileTxnSnapLog logFactory：通过dataDir和dataLogDir目录，用于事务日志记录和内存DataTree和session数据的快照。

-	Map<Long, QuorumServer> quorumPeers:QuorumServer包含ip、和Leader通信端口、选举端口即上述server.A=B:C:D的内容。而这里的key则是A,即server的id。这里的server不包含observers,即这里的server都是要参与投票的。

-	int electionType：选举算法的类型。默认是3，采用的是FastLeaderElection选举算法。如下图

	![创建选举算法的过程](https://static.oschina.net/uploads/img/201508/17070458_4lsW.png "创建选举算法的过程")

	目前前三种选举算法都被标记为过时了，只保留了最后一种选举算法。具体的选举过程，后面单独拿出一篇博客来分析。目前的首要目标是把集群的启动过程简单弄清楚，然后理解在集群时，如何来处理请求的整个过程。

-	long myid：就是本机器配置的id，即myid文件中写入的数字。

-	int tickTime、minSessionTimeout、maxSessionTimeout：这几个参数在单机版的时候都讲过了。

-	int initLimit、syncLimit：上面已经详细描述过了

-	QuorumVerifier quorumConfig：用于验证是否过半机器已经认同了。默认采用的是QuorumMaj，即最简单的数量过半即可，不考虑权重问题

-	ZKDatabase zkDb：即该服务器的内存数据库，最终还是会传递给ZooKeeperServer的子类。

-	LearnerType：就两种，PARTICIPANT, OBSERVER。PARTICIPANT参与投票，可能成为Follower，也可能成为Leader。OBSERVER不参与投票，角色不会改变。


然后就是启动QuorumPeer，之后阻塞主线程，启动过程如下：

![QuorumPeer启动过程](https://static.oschina.net/uploads/img/201508/17084516_gCVp.png "QuorumPeer启动过程")


主要分成4大步：

-	loadDataBase()：从事务日志目录dataLogDir和数据快照目录dataDir中恢复出DataTree数据

-	cnxnFactory.start()：开启对客户端的连接端口

-	startLeaderElection()：创建出选举算法

-	super.start()：启动QuorumPeer线程，在该线程中进行服务器状态的检查

不再重点说明选举过程，后面会专门抽出一篇博客来详细说明选举过程。

QuorumPeer本身继承了Thread，在run方法中不断的检测当前服务器的状态，即QuorumPeer的ServerState state属性。ServerState枚举内容如下：

	public enum ServerState {
        LOOKING, FOLLOWING, LEADING, OBSERVING;
    }

-	LOOKING：即该服务器处于Leader选举阶段

-	FOLLOWING：即该服务器作为一个Follower

-	LEADING:即该服务器作为一个Leader

-	OBSERVING：即该服务器作为一个Observer

在QuorumPeer的线程中操作如下：

-	服务器的状态是LOOKING，则根据之前创建的选举算法，执行选举过程

	选举过程一直阻塞，直到完成选举。完成选举后，各自的服务器根据投票结果判定自己是不是被选举成Leader了，如果不是则状态改变为FOLLOWING，如果是Leader，则状态改变为LEADING。

-	服务器的状态是LEADING：则会创建出LeaderZooKeeperServer服务器，然后封装成Leader，调用Leader的lead()方法，也是阻塞方法，只有当该Leader挂了之后，才去执行下setLeader(null)并重新回到LOOKING的状态

	![LEADING状态的操作](https://static.oschina.net/uploads/img/201508/17192344_IrLC.png "LEADING状态的操作")

-	服务器的状态是FOLLOWING：则会创建出FollowerZooKeeperServer服务器，然后封装成Follower，调用follower的followLeader()方法，也是阻塞方法，只有当该集群中的Leader挂了之后，才去执行下setFollower(null)并重新回到LOOKING的状态

	![FOLLOWING状态的操作](https://static.oschina.net/uploads/img/201508/17193633_14WL.png "FOLLOWING状态的操作")

下面就来详细的看看各个角色的启动过程：

##2.3 Leader和Follower启动过程

首先是根据已有的配置信息创建出LeaderZooKeeperServer：

![LeaderZooKeeperServer的创建](https://static.oschina.net/uploads/img/201508/17195445_cNd1.png "LeaderZooKeeperServer的创建")

然后就是封装成Leader对象

![封装成Leader对象](https://static.oschina.net/uploads/img/201508/17195754_2p5c.png "封装成Leader对象")

Leader和LeaderZooKeeperServer各自的职责是什么呢？

我们知道单机版使用的ZooKeeperServer不需要处理集群版中Follower与Leader之间的通信。ZooKeeperServer最主要的就是RequestProcessor处理器链、ZKDatabase、SessionTracker。这几部分是单机版和集群版服务器都共通的，主要不同的地方就是RequestProcessor处理器链的不同。所以LeaderZooKeeperServer、FollowerZooKeeperServer和ZooKeeperServer最主要的区别就是RequestProcessor处理器链。

集群版还要负责处理Follower与Leader之间的通信，所以需要在LeaderZooKeeperServer和FollowerZooKeeperServer之外加入这部分内容。所以就有了Leader对LeaderZooKeeperServer等封装，Follower对FollowerZooKeeperServer的封装。前者加上加入ServerSocket负责等待Follower的socket连接，后者加入Socket负责去连接Leader。

看下Leader处理socket连接的过程：

![Leader处理socket连接的过程](https://static.oschina.net/uploads/img/201508/18072002_aPsR.png "Leader处理socket连接的过程")

可以看到每来一个其他ZooKeeper服务器的socket连接，就会创建一个LearnerHandler，具体的处理逻辑就全部交给LearnerHandler了

然后在LearnerHandler中就开始了Leader和Follower或者Observer的初始化同步过程，这个过程之后详细讲解。完成同步之后，LearnerHandler就进行循环过程，不断的读取来自Follower或者Observer的数据包，如下：

	while (true) {
        qp = new QuorumPacket();
        ia.readRecord(qp, "packet");

        switch (qp.getType()) {
        case Leader.ACK:          
            break;
        case Leader.PING:          
            break;
        case Leader.REVALIDATE:        
            break;
        case Leader.REQUEST:                          
            break;
        default:
            LOG.warn("unexpected quorum packet, type: {}", packetToString(qp));
            break;
        }
    }

LearnerHandler会接收来自Follower或者Observer的PING、Request请求等。PING请求，则需要重新计算所传递过来的sessionId的过期时间。事务请求则需要Follower或者Observer转发给Leader，该事务请求就是Leader.REQUEST类型。

同时Follower也在不断接收来自Leader的数据包，处理如下：



Leader在开启与Follower或者Observer同步的时候，同时在启动了本身的RequestProcessor处理器链，如下：

![Leader的RequestProcessor处理器链](https://static.oschina.net/uploads/img/201508/18074408_kKAK.png "Leader的RequestProcessor处理器链")

PrepRequestProcessor-》ProposalRequestProcessor-》CommitProcessor-》ToBeAppliedRequestProcessor-》FinalRequestProcessor

ProposalRequestProcessor-》SyncRequestProcessor-》AckRequestProcessor

再来看看Follower的处理器链

![Follower的RequestProcessor处理器链](https://static.oschina.net/uploads/img/201508/18074933_sYeI.png "Follower的RequestProcessor处理器链")

FollowerRequestProcessor-》CommitProcessor-》FinalRequestProcessor

SyncRequestProcessor-》SendAckRequestProcessor

接下来就是需要详细的看看这些处理器链

##2.4 Leader和Follower的RequestProcessor处理器链

###2.4.1 Follower的FollowerRequestProcessor处理器

先来看下具体的处理过程：

![FollowerRequestProcessor处理过程](https://static.oschina.net/uploads/img/201508/18080216_Uuq4.png "FollowerRequestProcessor处理过程")

对于一个请求，先交给下一个处理器来处理，如果请求是事务请求，还要将该请求转发给Leader。zks.getFollower().request(request)即通过上述Leader与Follower的tcp连接发送给Leader，最终会在上述LearnerHandler中出现。

由于FollowerRequestProcessor的下一个处理器是CommitProcessor（是一个线程），nextProcessor.processRequest(request)这个操作仅仅是把request放入等待处理的队列中，然后就返回了，执行下面的代码，将事务请求转发给Leader。


#3 集群版建立连接过程