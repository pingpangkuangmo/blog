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

-	PrepRequestProcessor:对客户端用户发送过来的请求或者Follower、Observer转发过来的请求进行事务的区分，如果是事务请求则创建出事务请求头，后面的请求处理器就是依据是否有事务请求头来判断该请求是否是事务请求。同时进行一些验证工作如session是否过期，验证权限等操作。

-	ProposalRequestProcessor：主要对事务请求，向所有的Follower服务器发起一个议案，同时触发SyncRequestProcessor对事务请求的记录

-	SyncRequestProcessor：对事务请求记录到事务日志文件中，记录完成后触发AckRequestProcessor

-	AckRequestProcessor：对于上述议案，Leader也是投票的一份子，所以也要进行投票响应，只需执行下Leader的响应方法即可。而其他Follower服务器的投票响应则是需要向Leader发送一个Leader.ACK响应，Leader接收到后，同样去执行Leader的响应方法

在Leader的响应方法每执行一次，就会判断是否已经过半机器响应了，如果过半，则Leader向所有的Follower和Observer发送Leader.COMMIT请求，同时Leader也向自己的CommitProcessor中提交该事务请求，即该事务请求是通过过半机器认同的，需要被提交的事务请求。

-	CommitProcessor：对于已经被过半机器认同的请求交给下一个处理器处理。而那些还没有被过半机器认同的，则处于阻塞状态。

-	ToBeAppliedRequestProcessor：负责将请求交给下一个处理器FinalRequestProcessor，处理完毕后表示已经完成该项议案，然后就删除了该项议案

-	
##2.2 Follower服务器

##2.3 Observer服务器


#3 连接Leader建立session关联的过程和session不断激活的过程


#4 连接Follower建立session关联的过程和session不断激激活的过程
#5 连接Observer建立session关联的过程和session不断激激活的过程
#6 连接Leader执行setData的过程
#7 连接Follower执行setData的过程
#8 连接Observer执行setData的过程