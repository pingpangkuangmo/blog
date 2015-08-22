#1 系列目录

-	[ZooKeeper源码研究系列（1）源码环境搭建](http://my.oschina.net/pingpangkuangmo/blog/484955)
-	[ZooKeeper源码研究系列（2）客户端创建连接过程分析](http://my.oschina.net/pingpangkuangmo/blog/486780)
-	[ZooKeeper源码研究系列（3）单机版服务器介绍](http://my.oschina.net/pingpangkuangmo/blog/491673)
-	[ZooKeeper源码研究系列（4）集群版建立连接和更新数据过程](http://my.oschina.net/pingpangkuangmo/blog/495311)

#2 各服务器角色的请求处理器链

先介绍下Leader、Follower、Observer服务器的请求处理器链

##2.1 Leader服务器
	
PrepRequestProcessor-》ProposalRequestProcessor-》CommitProcessor-》ToBeAppliedRequestProcessor-》FinalRequestProcessor

ProposalRequestProcessor-》SyncRequestProcessor-》AckRequestProcessor

下面分别一一介绍

-	1 PrepRequestProcessor:首先为请求分配请求号zxid，然后对客户端用户发送过来的请求或者Follower、Observer转发过来的请求进行事务的区分，如果是事务请求则创建出事务请求头，后面的请求处理器就是依据是否有事务请求头来判断该请求是否是事务请求。同时进行一些验证工作如session是否过期，验证权限等操作。

-	2 ProposalRequestProcessor：主要对事务请求，向所有的Follower服务器发起一个议案，同时触发SyncRequestProcessor对事务请求的记录

-	3 SyncRequestProcessor：对事务请求记录到事务日志文件中，记录完成后触发AckRequestProcessor

-	4 AckRequestProcessor：对于上述议案，Leader也是投票的一份子，所以也要进行投票响应，只需执行下Leader的响应方法即可。而其他Follower服务器的投票响应则是需要向Leader发送一个Leader.ACK响应，Leader接收到后，同样去执行Leader的响应方法

在Leader的响应方法每执行一次，就会判断是否已经过半机器响应了，如果过半，则Leader向所有的Follower和Observer发送Leader.COMMIT请求，同时Leader也向自己的CommitProcessor中提交该事务请求，即该事务请求是通过过半机器认同的，需要被提交的事务请求。

-	5 CommitProcessor：对于已经被过半机器认同的请求交给下一个处理器处理。而那些还没有被过半机器认同的，则处于阻塞状态。

-	6 ToBeAppliedRequestProcessor：负责将请求交给下一个处理器FinalRequestProcessor，处理完毕后表示已经完成该项议案，然后就删除了该项议案

-	7 FinalRequestProcessor:最后一个请求处理器。对于事务请求，执行事务的具体操作，如增删改node、createSession、closeSession等。对于非事务操作如获取数据等，从DataTree中获取相应的数据。最终返回数据给客户端。

##2.2 Follower服务器

FollowerRequestProcessor-》CommitProcessor-》FinalRequestProcessor

SyncRequestProcessor-》SendAckRequestProcessor

-	1 FollowerRequestProcessor：首先将请求交给下一个处理器即CommitProcessor处理器，如果是该请求是事务请求或者前面有事务请求在等待处理，则该请求会被阻塞。如果是事务请求，交给CommitProcessor处理器之后，又立马将该请求转发给Leader，即事务请求必须要经过Leader，然后Leader又会把该事务请求封装成一个议案发给各个Follower服务器进行投票，各个Follower服务器接收到Leader发送过来议案后，首先要把这个议案请求记录到事务日志中，即调用SyncRequestProcessor来处理

-	2 CommitProcessor：一旦是事务请求，就需要等待该事务请求被过半数认可，接收到Leader的Leader.COMMIT请求，才会继续走下去

-	3 SyncRequestProcessor：把Leader发送过来的议案记录到事务日志中，然后交给下一个处理器SendAckRequestProcessor

-	4 SendAckRequestProcessor：当日志记录完成之后，需要给Leader发送一个Leader.ACK响应，表示已经成功记录在案。

Leader开始统计Follower发送过来的响应，一旦有过半机器发送过来响应，则认为该事务可以提交了。然后Leader就向所有的Follower发送Leader.COMMIT请求，带上之前的请求号，向所有的Observer发送Leader.INFORM请求，带上之前的整个请求内容，因为Follower在前面已经接收到了该请求，而Observer则没有接收到该请求，所以要对Observer带上整个请求内容。

Follower接收到Leader发送过来的Leader.COMMIT请求之后，根据带过来的请求号，找到真个请求对象，然后放到Follower的CommitProcessor中，使之继续走下去，交给FinalRequestProcessor

-	5 FinalRequestProcessor：最后一个请求处理器。同上面一样。对于事务请求，执行事务的具体操作，如增删改node、createSession、closeSession等。对于非事务操作如获取数据等，从DataTree中获取相应的数据。最终返回数据给客户端

##2.3 Observer服务器

ObserverRequestProcessor-》CommitProcessor-》FinalRequestProcessor

SyncRequestProcessor

其中上述SyncRequestProcessor是通过配置zookeeper.observer.syncEnabled系统属性的true or false来决定是否需要这个处理器，默认true

-	ObserverRequestProcessor:和FollowerRequestProcessor功能完全一样，只是参数不一样，可以抽象出来的。

所以我们看到Observer服务器和Follower服务器的处理器链基本差不多。不同之处就是Follower服务器还有一个SendAckRequestProcessor，向Leader发送投票反馈。而Observer不参与投票，则不需要这个处理器。


以上就大致说完了各个服务器角色的请求处理器链，下面就结合具体的请求案例，再来捋一下整个过程

#3 连接Leader建立session关联的过程和session不断激活的过程

这里连接的服务器以Leader为例

这就需要从用户创建ZooKeeper对象开始说起。

-	1 用户创建ZooKeeper对象，内部创建出ClientCnxn，可以简单想象成ZooKeeper对象的内部管家，ClientCnxn有两个主要的线程SendThread和EventThread

	SendThread负责与服务器端的通信，EventThread负责事件的通知。

	-	1.1 SendThread启动之后，就从创建ZooKeeper对象的地址列表（被随机打乱了），取出一个服务器地址进行tcp连接操作

	-	1.2 当tcp连接连接成功之后，就需要和服务器端建立session关联。依托tcp连接，向服务器端发送ConnectRequest请求，会把创建ZooKeeper对象时指定的sessionTimeout时间带上

-	2 客户端一旦和服务器端建立tcp连接之后，服务器端会给客户端创建一个ServerCnxn，专门负责与该客户端的通信

	-	2.1 当客户端第一次发送ConnectRequest请求到ServerCnxn中，ServerCnxn首先会对tcp连接传递过来的数据序列化成ConnectRequest，拿到客户端传递的sessionTimeout时间，由于服务器端在启动的时候指定了maxSessionTimeout、minSessionTimeout（即使没有指定，也会使用默认的），要求客户端传递过来的sessionTimeout时间必须在此两者之间，不符合要求的分别取对应的最大值或者最小值
	
	-	2.2 然后就使用Leader服务器的SessionTracker（session管理器）根据上面协商后的sessionTimeout时间，分配出sessionId，创建出session

	-	2.3 根据分配的sessionId和刚才的ServerCnxn创建出一个请求，类型为OpCode.createSession，将该请求提交到Leader的请求处理器链上

	-	2.4 首先遇到的是PrepRequestProcessor处理器，认为OpCode.createSession请求是一个事务请求，就创建了一个事务请求体，
#4 连接Follower建立session关联的过程和session不断激激活的过程
#5 连接Observer建立session关联的过程和session不断激激活的过程
#6 连接Leader执行setData的过程
#7 连接Follower执行setData的过程
#8 连接Observer执行setData的过程