# 1 kafka基础

本篇文章讨论的kafka版本是目前最新版 0.10.1.0。

## 1.1 kafka种的KafkaController

所有broker会通过ZooKeeper选举出一个作为KafkaController，来负责：

-	分区的状态维护：负责分区的新增、下线等，分区副本的leader选举
-	副本的状态维护：负责副本的新增、下线等

## 1.2 kafka分区中的基本概念

每个分区可以有多个副本，分散在不同的broker上。

-	leader副本：被KafkaController选举出来的，作为该分区的leader
-	其他follower副本：其他副本都作为follower副本
-	isr列表：简单描述就是，"跟得上"leader的副本列表（包含leader）,最开始是所有副本。这里的跟得上是指

	-	replica.lag.time.max.ms：在0.9.0.0之前表示follower如果在此时间间隔内没有向leader发送fetch请求，则该follower就会被剔除isr列表，在0.9.0.0之后表示如果该follower在此时间间隔内一直没有追上过leader的所有消息，则该follower就会被剔除isr列表
	-	replica.lag.max.messages（0.9.0.0版本中已被废除）：follower如果落后leader的消息个数超过该值，则该follower就会被剔除isr列表
		废除的主要原因是：目前这个配置是个统一配置，不同的topic速率生产速率不太一样，没办法来指定一个具体的值来应用到所有的topic上。将来可以将这个配置下放到topic级别，关于这个问题，可以见这里的讨论[Automate replica lag tuning](https://issues.apache.org/jira/browse/KAFKA-1546)

每一个producer发送消息给某个分区的leader副本，其他follower副本又会复制该消息。producer端有一个acks参数可以设置：

-	acks=0：表示producer不需要leader发送响应，即producer只管发不管发送成功与否。延迟低，容易丢失数据。
-	acks=1：表示leader写入成功（但是并没有刷新到磁盘）后即向producer响应。延迟中等，一旦leader副本挂了，就会丢失数据。
-	acks=-1：表示leader会等待isr列表中所有副本都写入成功才向producer发送响应。延迟高、可靠性高。但是也会丢数据，下面会详细讨论

同时对于isr列表的数量要求也有一个配置

-	min.insync.replicas：默认是1。当acks=-1的时候，leader在处理新消息前，会先判断当前isr列表的的size是否小于这个值，如果小于的话，则不允许写入，返回NotEnoughReplicasException异常。同时，一旦允许写入了之后，在响应producer之前也会判断当前isr列表的size是否小于该值，如果小于返回NotEnoughReplicasAfterAppendException异常

我们本篇文章就重点通过kafka的原理来揭示在acks=-1的情况下，哪些情况下会丢失数据，或许可以提一些改进措施来做到不丢失数据。

下面会先介绍下leader和follower副本复制的原理

## 1.3 副本复制过程

-	leader副本的属性

	-	highWatermarkMetadata：代表已经被isr列表复制的最大offset，consumer只能消费该offset之前的数据
	-	logEndOffsetMetadata：代表leader副本上已经复制的最大offset

	leader副本拥有其他副本的记录，保存着他们的如下属性：

	-	logEndOffsetMetadata：代表该follower副本已经复制的最大offset
	-	lastCaughtUpTimeMs：记录该follower副本上一次追上leader副本的所有消息的时间

-	follower副本的属性

	-	highWatermarkMetadata：follower会获取到leader的highWatermarkMetadata更新到自己的该属性中
	-	logEndOffsetMetadata：代表follower副本上已经复制的最大offset

	其中follower会不断的向leader发送fetch请求，如果没有数据fetch则被leader阻塞一段时间，等待新数据的来临，一旦来临则解除阻塞，复制数据给follower。

我们来看下当acks=-1时，一次消息写入的整个过程，上述是属性是怎么变化的

-	1.3.1 消息准备写入leader副本，leader副本首先判断当前isr列表是否小于min.insync.replicas，不小于才允许写入。

	如果不小于，leader写入到自己的log中，得到该消息的offset，然后对其他follower的fetch请求解除阻塞，复制一定量的消息给follower

	同时leader将自己最新的highWatermarkMetadata传给follower

	同时会判断这次复制是否复制到leader副本的末尾了，即logEndOffsetMetadata位置，如果是的话，则更新上述的lastCaughtUpTimeMs

-	1.3.2 follower会将fetch来的数据写入到自己的log中，自己的logEndOffsetMetadata得到了更新，同时更新自己的highWatermarkMetadata，就是取leader传来的highWatermarkMetadata和自己的logEndOffsetMetadata中的最小值

	然后follower再一次向leader发送fetch请求，fetch的初始offset就是自己的logEndOffsetMetadata+1。

-	1.3.3 leader副本收到该fetch后，会更新leader副本中该follower的logEndOffsetMetadata为上述fetch的offset，同时会对所有的isr列表的logEndOffsetMetadata排序得到最小的logEndOffsetMetadata作为最新的highWatermarkMetadata

	如果highWatermarkMetadata已经大于了leader写入该消息的offset了，说明该消息已经被isr列表都复制过了，则leader开始回应producer

	判断当前isr列表的size是否小于min.insync.replicas，如果小于返回NotEnoughReplicasAfterAppendException异常，不小于则代表正常写入了。

-	1.3.4 follower在下一次的fetch请求的响应中就会得到leader最新的highWatermarkMetadata，更新自己的highWatermarkMetadata

## 1.4 leader副本选举

如果某个broker挂了，leader副本在该broker上的分区就要重新进行leader选举。来简要描述下leader选举的过程

-	1.4.1 KafkaController会监听ZooKeeper的/brokers/ids节点路径，一旦发现有broker挂了，执行下面的逻辑。这里暂时先不考虑KafkaController所在broker挂了的情况，KafkaController挂了，各个broker会重新leader选举出新的KafkaController

-	1.4.2 leader副本在该broker上的分区就要重新进行leader选举，目前的选举策略是

	-	1.4.2.1 优先从isr列表中选出第一个作为leader副本
	-	1.4.2.2 如果isr列表为空，则查看该topic的unclean.leader.election.enable配置。

		unclean.leader.election.enable：为true则代表允许选用非isr列表的副本作为leader，那么此时就意味着数据可能丢失，为false的话，则表示不允许，直接抛出NoReplicaOnlineException异常，造成leader副本选举失败。
	-	1.4.2.3 如果上述配置为true，则从其他副本中选出一个作为leader副本，并且isr列表只包含该leader副本。

	一旦选举成功，则将选举后的leader和isr和其他副本信息写入到该分区的对应的zk路径上。

-	1.4.3 KafkaController向上述相关的broker上发送LeaderAndIsr请求，将新分配的leader、isr、全部副本等信息传给他们。同时将向所有的broker发送UpdateMetadata请求，更新每个broker的缓存的metadata数据。

-	1.4.4 

# 2 消息丢失

哪些场景下会丢失消息？

-	即使设置acks=-1，但是isr列表为空，只能选择其他存活的副本。
-	即使设置acks=-1，并且选出isr中的副本作为leader的时候，仍然是会存在丢数据的情况的：

	s1 s2 s3是isr列表，s1 leader，一旦某个日志写入到s1 s2 s3，则s1将hw提高，并回复了客户端ok，但是s2 s3的hw可能还没被更新，此时s1挂了，s2当选leader了，s2的日志不变，但是s3就要截断日志了，这时已经回复客户端的日志是没有丢的，因为s2种已经复制了。

	但是如果此时s2一旦挂了，s3当选，则s3上就不存在上述日志了。这时候就造成日志丢失了。最主要的是这个截断操作是可能截断了已提交的日志，这样的话，这部分日志就在其他副本中消失了。

	那zab和raft有没有这种问题呢？他们是如何解决和规避的呢？

	raft并不进行日志的截断操作，而是会通过不断的一致性检查来实现和leader来保持一致。要么和leader保持一致，要么还是自己原来的日志，不会出现截断已提交的日志。

	ZooKeeper只有当某个follower的记录超出leader的部分才会截断，其他的不会截断的。所以也不会出现日志丢失的问题。

# 4 顺序性



# 5 相关场景


# 6 讨论

-	1 kafka为什么不采用过半确认？

	ZooKeeper过半确认的好处是？
	主要是为了leader出现故障的时候，保证已经写入的日志存在与过半的机器中，因此在新的leader选举中，可以保证不丢失已提交的日志。
	
	那kafka什么情况下丢失？能不能做到不丢失？
	
	如果总副本数是3，设置minIsr=2（默认是1）,则意味着：

	-	在写入的时候，如果isr列表数小于2，则无法写入，算是牺牲可用性来保证一致性吧
	-	在producer写入之后等待响应的时候，也会检查是否小于minIsr，小于的话则NOT_ENOUGH_REPLICAS_AFTER_APPEND
	
	


-	2 minIsr在什么时候用？
-	3 kafka是分区内有序的，zookeeper是全局有序的，因为kafka是多写入，而zookeeper是单写入的
-	4 截断操作是删除了不需要的日志吗？
-	5 截断到hw，保证了顺序性，但是也造成了丢失日志的可能，能不能换一种解决方案呢？
-	6 kafka的isr和过半怎么对比呢？你会怎么选？

	

**欢迎继续来讨论，越辩越清晰**。

欢迎关注微信公众号：乒乓狂魔

![乒乓狂魔微信公众号](https://static.oschina.net/uploads/img/201610/28090041_LsUp.png "乒乓狂魔微信公众号")