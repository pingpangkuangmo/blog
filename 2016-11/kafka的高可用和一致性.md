# 1 kafka基础

本篇文章讨论的kafka版本是目前最新版 0.10.1.0。

## 1.1 kafka种的KafkaController

所有broker会通过ZooKeeper选举出一个作为KafkaController，来负责：

-	监控所有broker的存活，以及向他们发送相关的执行命令。
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

我们来看下当producer的acks=-1时，一次消息写入的整个过程，上述是属性是怎么变化的

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

-	1.4.4 如果是leader副本，更新该分区的leader、isr、所有副本等信息。如果自己之前就是leader，则现在什么操作都不用做。如果之前不是leader，则需将自己保存的所有follower副本的logEndOffsetMetadata设置为UnknownOffsetMetadata，之后等待follower的fetch，就会进行更新
-	1.4.5 如果是follower副本，更新该分区的leader、isr、所有副本等信息

	**然后将日志截断到自己保存的highWatermarkMetadata位置，即日志的logEndOffsetMetadata等于了highWatermarkMetadata**

	最后创建新的fetch请求线程，向新leader不断发送fetch请求，初次fetch的offset是logEndOffsetMetadata。

**上述重点就是leader副本的日志不做处理，而follower的日志则需要截断到highWatermarkMetadata位置。**

至此，算是简单描述了分区的基本情况，下面就针对上述过程来讨论下kafka分区的高可用和一致性问题。

# 2 消息丢失

## 2.1 消息丢失的场景

哪些场景下会丢失消息？

-	acks= 0、1，很明显都存在消息丢失的可能。

-	即使设置acks=-1，当isr列表为空，如果unclean.leader.election.enable为true，则会选择其他存活的副本作为新的leader，也会存在消息丢失的问题。

-	即使设置acks=-1，当isr列表为空，如果unclean.leader.election.enable为false，则不会选择其他存活的副本作为新的leader，即牺牲了可用性来防止上述消息丢失问题。

-	即使设置acks=-1，并且选出isr中的副本作为leader的时候，仍然是会存在丢数据的情况的：

	s1 s2 s3是isr列表，还有其他副本为非isr列表，s1是leader，一旦某个日志写入到s1 s2 s3，则s1将highWatermarkMetadata提高，并回复了客户端ok，但是s2 s3的highWatermarkMetadata可能还没被更新，此时s1挂了，s2当选leader了，s2的日志不变，但是s3就要截断日志了，这时已经回复客户端的日志是没有丢的，因为s2已经复制了。

	但是如果此时s2一旦挂了，s3当选，则s3上就不存在上述日志了（前面s2当选leader的时候s3已经将日志截断了），这时候就造成日志丢失了。

## 2.2 不丢消息的探讨

其实我们是希望上述最后一个场景能够做到不丢消息的，但是目前的做法还是可能会丢消息的。

丢消息最主要的原因是：

**由于follower的highWatermarkMetadata相对于leader的highWatermarkMetadata是延迟更新的，当leader选举完成后，所有follower副本的截断到自己的highWatermarkMetadata位置，则可能截断了已被老leader提交了的日志，这样的话，这部分日志仅仅存在新的leader副本中，在其他副本中消失了，一旦leader副本挂了，这部分日志就彻底丢失了**

这个截断到highWatermarkMetadata的操作的确太狠了，但是它的用途有一个就是：**避免了日志的不一致的问题**。通过每次leader选举之后的日志截断，来达到和leader之间日志的一致性，避免出现日志错乱的情况。

ZooKeeper和Raft的实现也有类似的日志复制的问题，那ZooKeeper和Raft的实现有没有这种问题呢？他们是如何解决的呢？

Raft并不进行日志的截断操作，而是会通过每次日志复制时的一致性检查来进行日志的纠正，达到和leader来保持一致的目的。不截断日志，那么对于已经提交的日志，则必然存在过半的机器上从而能够保证日志基本是不会丢失的。

ZooKeeper只有当某个follower的记录超出leader的部分才会截断，其他的不会截断的。选举出来的leader是经过过半pk的，必然是包含全部已经被提交的日志的，即使该leader挂了，再次重新选举，由于不进行日志截断，仍然是可以选出其他包含全部已提交的日志的（有过半的机器都包含全部已提交的日志）。ZooKeeper对于日志的纠正则是在leader选举完成后专门开启一个纠正过程。


kafka的截断到highWatermarkMetadata的确有点太粗暴了，如果不截断日志，则需要解决日志错乱的问题，即使不能够像ZooKeeper那样花大代价专门开启一个纠正过程，可以像Raft那样每次在fetch的时候可以进行不断的纠正。这一块还有待继续关注。

# 3 顺序性

kafka目前是只能保证一个分区内的数据是有序的。

但是你可能经常听说，一旦某个broker挂了，就可能产生乱序问题（也没人指出乱序的原因），是否正确呢？

首先来看看如何能保证单个分区内消息的有序性，有如下几个过程：

-	3.1 producer按照消息的顺序进行发送

	很多时候为了发送效率，采用的办法是多线程、异步、批量发送。

	如果为了保证顺序，则不能使用多线程来执行发送任务。

	异步：一般是把消息先发到一个队列中，由后台线程不断的执行发送任务。这种方式对消息的顺序也是有影响的：

	如先发送消息1，后发送消息2，此时服务器端在处理消息1的时候返回了异常，可能在处理消息2的时候成功了，此时若再重试消息1就会造成消息乱序的问题。所以producer端需要先确认消息1发送成功了才能执行消息2的发送。

	对于kafka来说，目前是异步、批量发送。解决异步的上述问题就是配置如下属性：

		max.in.flight.requests.per.connection=1

	即producer发现一旦还有未确认发送成功的消息，则后面的消息不允许发送。

-	3.2 相同key的消息能够hash到相同的分区

	正常情况下是没问题的，但是一旦某个分区挂了，如原本总共4个分区，此时只有3个分区存活，在此分区恢复的这段时间内，是否会存在hash错乱到别的分区？

	那就要看producer端获取的metadata信息是否会立马更新成3个分区。目前看来应该是不会的

	producer见到的metadata数据是各个broker上的缓存数据，这些缓存数据是由KafkaController来统一进行更新的。一旦leader副本挂了，KafkaController并不会去立马更新成3个分区，而是去执行leader选举，选举完成后才会去更新metadata数据，此时选举完成后仍然是保证4个分区的，也就是说producer是不可能获取到只有3个分区的metadata数据的，所以producer端还是能正常hash的，不会错乱分区的。

	在整个leader选举恢复过程，producer最多是无法写入数据（后期可以重试）。

-	3.3 系统对顺序消息的支持

	leader副本按照消息到来的先后顺序写入本地日志的，一旦写入日志后，该顺序就确定了，follower副本也是按照该顺序进行复制的。对于消息的提交也是按照消息的offset从低到高来确认提交的，所以说kafka对于消息的处理是顺序的。

-	3.4 consumer能够按照消息的顺序进行消费

	为了接收的效率，可能会使用多线程进行消费。这里为了保证顺序就只能使用单线程来进行消费了。

	目前kafka的Consumer有scala版本的和java版本的（这一块之后再详细探讨），最新的java版本，对用户提供一个poll方法，用户自己去决定是使用多线程还是单线程。

# 4 其他话题

-	如何看待kafka的isr列表设计？和过半怎么对比呢？

	对于相同数量的2n个follower加一个leader，过半呢则允许n个follower挂掉，而isr呢则允许2n个follower挂掉（但是会存在丢失消息的问题），所以过半更多会牺牲可用性（挂掉一半以上就不可用了）来增强数据的一致性，而isr会牺牲一致性来增强可用性（挂掉一半以上扔可使用，但是存在丢数据的问题）

	但是在确认效率上：过半仅仅需要最快的n+1的写入成功即可判定为成功，而isr则需要2n+1的写入成功才算成功。同时isr是动态变化的过程，一旦跟不上或者跟上了都会离开或者加入isr列表。isr列表越小写入速度就会加快。

-	有哪些环节会造成消息的重复消费？如果避免不了，如何去减少重复？

	-	producer端重复发送

		producer端因发送超时等等原因做重试操作，目前broker端做重复请求的判断还是很难的，目前kafka也没有去做，而是存储完消息之后，如果开启了Log compaction，它会通过kafka消息中的key来判定是否是重复消息，是的话则会删除。

	-	consumer消费后，未及时提交消费的offset便挂了，下次恢复后就会重复消费

		这个目前来说并没有通用的解决办法，先消费后提交offset可能会重复，先提交offset后消费可能造成消息丢失，所以一般还是优先保证消息不丢，在业务上去做去重判断。

本文章涉及到的话题很多，难免有疏漏的地方，还请批评指正。

**欢迎继续来讨论，越辩越清晰**。

欢迎关注微信公众号：乒乓狂魔

![乒乓狂魔微信公众号](https://static.oschina.net/uploads/img/201610/28090041_LsUp.png "乒乓狂魔微信公众号")