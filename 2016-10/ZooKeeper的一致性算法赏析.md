# 1 ZAB介绍

ZAB协议全称就是ZooKeeper Atomic Broadcast protocol，是ZooKeeper用来实现一致性的算法，分成如下4个阶段。

先来解释下部分名词

electionEpoch：每执行一次leader选举，electionEpoch就会自增，用来标记leader选举的轮次

peerEpoch：每次leader选举完成之后，都会选举出一个新的peerEpoch，用来标记事务请求所属的轮次

zxid：事务请求的唯一标记，由leader服务器负责进行分配。由2部分构成，高32位是上述的peerEpoch，低32位是请求的计数，从0开始。所以由zxid我们就可以知道该请求是哪个轮次的，并且是该轮次的第几个请求。

lastProcessedZxid：最后一次commit的事务请求的zxid

-	Leader election

	leader选举过程，electionEpoch自增，在选举的时候lastProcessedZxid越大，越有可能成为leader

-	Discovery：

	第一：leader收集follower的lastProcessedZxid，这个主要用来通过和leader的lastProcessedZxid对比来确认follower需要同步的数据范围

	第二：选举出一个新的peerEpoch，主要用于防止旧的leader来进行提交操作（旧leader向follower发送命令的时候，follower发现zxid所在的peerEpoch比现在的小，则直接拒绝，防止出现不一致性）

-	Synchronization：

	follower中的事务日志和leader保持一致的过程，就是依据follower和leader之间的lastProcessedZxid进行，follower多的话则删除掉多余部分，follower少的话则补充，一旦对应不上则follower删除掉对不上的zxid及其之后的部分然后再从leader同步该部分之后的数据

-	Broadcast

	正常处理客户端请求的过程。leader针对客户端的事务请求，然后提出一个议案，发给所有的follower，一旦过半的follower回复OK的话，leader就可以将该议案进行提交了，向所有follower发送提交该议案的请求，leader同时返回OK响应给客户端

上面简单的描述了上述4个过程，这4个过程的详细描述在zab的paper中可以找到，但是我看了之后基本和zab的源码实现上相差有点大，这里就不再提zab paper对上述4个过程的描述了，下面会详细的说明ZooKeeper源码中是具体怎么来实现的

# 2 ZAB协议源码实现

先看下ZooKeeper整体的实现情况，如下图所示

![zab实现](https://static.oschina.net/uploads/img/201610/31160806_blP5.png "zab实现")

上述实现中Recovery Phase包含了ZAB协议中的Discovery和Synchronization。

## 2.1 重要的数据介绍

加上前面已经介绍的几个名词

-	long lastProcessedZxid：最后一次commit的事务请求的zxid
-	LinkedList<Proposal\> committedLog、long maxCommittedLog、long minCommittedLog：

	ZooKeeper会保存最近一段时间内执行的事务请求议案，个数限制默认为500个议案。上述committedLog就是用来保存议案的列表，上述maxCommittedLog表示最大议案的zxid，minCommittedLog表示committedLog中最小议案的zxid。

-	ConcurrentMap<Long, Proposal> outstandingProposals

	Leader拥有的属性，每当提出一个议案，都会将该议案存放至outstandingProposals，一旦议案被过半认同了，就要提交该议案，则从outstandingProposals中删除该议案

-	ConcurrentLinkedQueue<Proposal> toBeApplied

	Leader拥有的属性，每当准备提交一个议案，就会将该议案存放至该列表中，一旦议案应用到ZooKeeper的内存树中了，然后就可以将该议案从toBeApplied中删除


对于上述几个参数，整个Broadcast的处理过程可以描述为：

-	leader针对客户端的事务请求（leader为该请求分配了zxid），创建出一个议案，并将zxid和该议案存放至leader的outstandingProposals中
-	leader开始向所有的follower发送该议案，如果过半的follower回复OK的话，则leader认为可以提交该议案，则将该议案从outstandingProposals中删除，然后存放到toBeApplied中
-	leader对该议案进行提交，会向所有的follower发送提交该议案的命令，leader自己也开始执行提交过程，会将该请求的内容应用到ZooKeeper的内存树中，然后更新lastProcessedZxid为该请求的zxid，同时将该请求的议案存放到上述committedLog，同时更新maxCommittedLog和minCommittedLog
-	leader就开始向客户端进行回复，然后就会将该议案从toBeApplied中删除

## 2.2 Fast Leader Election

leader选举过程要关注的要点：

-	所有机器刚启动时进行leader选举过程
-	如果leader选举完成，刚启动起来的server怎么识别到leader选举已完成

投票过程有3个重要的数据:

-	ServerState

	目前ZooKeeper机器所处的状态有4种，分别是

	-	LOOKING：进入leader选举状态
	-	FOLLOWING：leader选举结束，进入follower状态
	-	LEADING：leader选举结束，进入leader状态
	-	OBSERVING：处于观察者状态

-	HashMap<Long, Vote> recvset

	用于收集LOOKING、FOLLOWING、LEADING状态下的server的投票

-	HashMap<Long, Vote> outofelection

	用于收集FOLLOWING、LEADING状态下的server的投票（能够收集到这种状态下的投票，说明leader选举已经完成）

下面就来详细说明这个过程：

-	1 serverA首先将electionEpoch自增，然后为自己投票

	serverA会首先从快照日志和事务日志中加载数据，就可以得到本机器的内存树数据，以及lastProcessedZxid（这一部分后面再详细说明）

	初始投票Vote的内容：

	-	proposedLeader：ZooKeeper Server中的myid值，初始为本机器的id
	-	proposedZxid：最大事务zxid，初始为本机器的lastProcessedZxid
	-	proposedEpoch:peerEpoch值，由上述的lastProcessedZxid的高32得到

	然后该serverA向其他所有server发送通知，通知内容就是上述投票信息和electionEpoch信息

-	2 serverB接收到上述通知，然后进行投票PK

	如果serverB收到的通知中的electionEpoch比自己的大，则serverB更新自己的electionEpoch为serverA的electionEpoch

	如果该serverB收到的通知中的electionEpoch比自己的小，则serverB向serverA发送一个通知，将serverB自己的投票以及electionEpoch发送给serverA，serverA收到后就会更新自己的electionEpoch

	在electionEpoch达成一致后，就开始进行投票之间的pk，规则如下：

		/*
         * We return true if one of the following three cases hold:
         * 1- New epoch is higher
         * 2- New epoch is the same as current epoch, but new zxid is higher
         * 3- New epoch is the same as current epoch, new zxid is the same
         *  as current zxid, but server id is higher.
         */
        
        return ((newEpoch > curEpoch) || 
                ((newEpoch == curEpoch) &&
                ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));

	就是优先比较proposedEpoch，然后优先比较proposedZxid，最后优先比较proposedLeader

	pk完毕后，如果本机器投票被pk掉，则更新投票信息为对方投票信息，同时重新发送该投票信息给所有的server。

	如果本机器投票没有被pk掉，则看下面的过半判断过程
	
-	3 根据server的状态来判定leader

	如果当前发来的投票的server的状态是LOOKING状态，则只需要判断**本机器的投票**是否在recvset中过半了，如果过半了则说明leader选举就算成功了，如果当前server的id等于上述过半投票的proposedLeader,则说明自己将成为了leader，否则自己将成为了follower

	如果当前发来的投票的server的状态是FOLLOWING、LEADING状态，则说明leader选举过程已经完成了，则**发过来的投票就是leader的信息**，这里就需要**判断发过来的投票**是否在recvset或者outofelection中过半了

	同时还要检查leader是否给自己发送过投票信息，从投票信息中确认该leader是不是LEADING状态（这一部分还需要仔细推敲下）


## 2.3 Recovery Phase

一旦leader选举完成，就开始进入恢复阶段，就是follower要同步leader上的数据信息

-	1 通信初始化

	leader会创建一个ServerSocket，接收follower的连接，leader会为每一个连接会用一个LearnerHandler线程来进行服务

-	2 重新为peerEpoch选举出一个新的peerEpoch

	follower会向leader发送一个Leader.FOLLOWERINFO信息，包含自己的peerEpoch信息

	leader的LearnerHandler会获取到上述peerEpoch信息，从中选出一个最大的peerEpoch，然后加1作为新的peerEpoch。

	然后leader的所有LearnerHandler会向各自的follower发送一个Leader.LEADERINFO信息，包含上述新的peerEpoch

	follower会使用上述peerEpoch来更新自己的peerEpoch，同时将自己的lastProcessedZxid发给leader

	leader的所有LearnerHandler会记录上述各自follower的lastProcessedZxid，然后根据这个lastProcessedZxid和leader的lastProcessedZxid之间的差异进行同步

-	3 已经处理的事务议案的同步

	判断LearnerHandler中的lastProcessedZxid是否在minCommittedLog和maxCommittedLog之间

	-	LearnerHandler中的lastProcessedZxid和leader的lastProcessedZxid一致，则说明已经保持同步了
	-	如果lastProcessedZxid在minCommittedLog和maxCommittedLog之间

		从lastProcessedZxid开始到maxCommittedLog结束的这部分议案，重新发送给该LearnerHandler对应的follower，同时发送对应议案的commit命令

		上述可能存在一个问题：即lastProcessedZxid虽然在他们之间，但是并没有找到lastProcessedZxid对应的议案，即这个zxid是leader所没有的，此时的策略就是完全按照leader来同步，删除该follower这一部分的事务日志，然后重新发送这一部分的议案，并提交这些议案
	-	如果lastProcessedZxid大于maxCommittedLog

		则删除该follower大于部分的事务日志
	-	如果lastProcessedZxid小于minCommittedLog

		则直接采用快照的方式来恢复

-	4 未处理的事务议案的同步

	LearnerHandler还会从leader的toBeApplied数据中将大于该LearnerHandler中的lastProcessedZxid的议案进行发送和提交（toBeApplied是已经被确认为提交的）

	LearnerHandler还会从leader的outstandingProposals中大于该LearnerHandler中的lastProcessedZxid的议案进行发送，不提交（outstandingProposals是还没被被确认为提交的）

-	5 将LearnerHandler加入到正式follower列表中

	意味着该LearnerHandler正式接受请求。即此时leader可能正在处理客户端请求，leader针对该请求发出一个议案，然后对该正式follower列表才会进行执行发送工作。这里有一个地方就是：

	上述我们在比较lastProcessedZxid和minCommittedLog和maxCommittedLog差异的时候，必须要获取leader内存数据的读锁，即在此期间不能执行修改操作，当欠缺的数据包已经补上之后（先放置在一个队列中，异步发送），才能加入到正式的follower列表，否则就会出现顺序错乱的问题

	同时也说明了，一旦一个follower在和leader进行同步的过程（这个同步过程仅仅是确认要发送的议案，先放置到队列中即可等待异步发送，并不是说必须要发送过去），该leader是暂时阻塞一切写操作的。

	对于快照方式的同步，则是直接同步写入的，写入期间对数据的改动会放在上述队列中的，然后当同步写入完成之后，再启动对该队列的异步写入。

	上述的要理解的关键点就是：既要不能漏掉，又要保证顺序

-	6 LearnerHandler发送Leader.NEWLEADER以及Leader.UPTODATE命令

	该命令是在同步结束之后发的，follower收到该命令之后会执行一次版本快照等初始化操作，如果收到该命令的ACK则说明follower都已经完成同步了并完成了初始化

	leader开始进入心跳检测过程，不断向follower发送心跳命令，不断检是否有过半机器进行了心跳回复，如果没有过半，则执行关闭操作，开始进入leader选举状态

	LearnerHandler向对应的follower发送Leader.UPTODATE，follower接收到之后，开始和leader进入Broadcast处理过程

## 2.4 Broadcast Phase

前面其实已经说过了，参见2.1中的内容

# 3 特殊情况的注意点

## 3.1 事务日志和快照日志的持久化

-	Broadcast过程的持久化

	leader针对每次事务请求都会生成一个议案，然后向所有的follower发送该议案

	follower接收到该议案后，所做的操作就是将该议案记录到事务日志中，每当记满100000个，则事务日志执行flush操作，同时开启一个新的文件来记录事务日志

	同时会执行内存树的快照，snapshot.[lastProcessedZxid]作为文件名创建一个新文件，快照内容保存到该文件中

-	leader shutdown过程的持久化

	一旦leader过半的心跳检测失败，则执行shutdown方法，在该shutdown中会对事务日志进行flush操作


## 3.2 重新选举leader时对之前数据的处理

### 3.2.1 重新选举前

造成重新选举有如下2种情况

-	Leader挂了

-	Leader得不到半数的follower支持


### 3.2.2 重新选举

raft对于entry是按照Index来进行处理的，一旦开启一个新的leader，新leader是需要处理前一个term的未提交的entry的
而zookeeper是按照epoch+index来进行处理的，一旦开启一个新的leader，新leader是不需要处理前一个term的未提交的entry的

## 3.3 follower挂了之后又重启的恢复过程

## 3.4 同步follower失败的情况

## 3.5 对client端是否一致

客户端收到OK回复，会不会丢失数据？
客户端没有收到OK回复，会不会多存储数据？

欢迎关注微信公众号：乒乓狂魔

![乒乓狂魔微信公众号](https://static.oschina.net/uploads/img/201610/28090041_LsUp.png "乒乓狂魔微信公众号")