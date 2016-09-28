# 1 系列

-	[整体架构图]()
-	producer端的设计
-	broker端接收消息
-	broker端消息的存储
-	consumer端的设计
-	consumer消费消息
-	分布式事务的实现
-	定时消息的实现

# 2 整体架构图


## 2.1 整体概念

先来看下官方给出的整体架构图

![RocketMQ的架构图](https://static.oschina.net/uploads/img/201609/28105945_t8eA.png "RocketMQ的架构图")

-	Producer集群：拥有相同的producerGroup,一般来讲，Producer不必要有集群的概念，这里的集群仅仅在RocketMQ的分布式事务中有用到

-	Name Server集群：提供topic的路由信息，路由信息数据存储在内存中，broker会定时的发送更新信息到nameserver中的每一个机器，来进行更新，所以name server集群可以简单理解为无状态（实际情况下可能会存在每个nameserver机器上的数据有短暂的不一致现象，但是通过定时更新，大部分情况下都是一致的）

-	broker集群：一个集群有一个统一的名字，即brokerClusterName，默认是DefaultCluster。一个集群下有多个master，每个master下有多个slave。master和slave算是一组，拥有相同的brokerName,不同的brokerId，master的brokerId是0，而slave则是大于0的值。master和slave之间可以进行同步复制或者是异步复制。

-	consumer集群：拥有相同的consumerGroup。

下面来说说他们至今的通信关系

-	Producer和Name Server：每一个Producer会与Name Server集群中的一台机器建立TCP连接，会从这台Name Server上拉取路由信息。
-	Producer和broker:Producer会和它要发送的topic相关的master类型的broker建立TCP连接，用于发送消息以及定时的心跳信息。broker中会记录该Producer的信息，供查询使用

-	broker与Name Server:broker（不管是master还是slave）会和每一台Name Server机器来建立TCP连接。broker在启动的时候会注册自己配置的topic信息到Name Server集群的每一台机器中。即每一台Name Server都有该broker的topic的配置信息。master与master之间无连接，master与slave之间有连接

-	Consumer和Name Server：每一个Consumer会和Name Server集群中的一台机器建立TCP连接，会从这台Name Server上拉取路由信息，进行负载均衡。

## 2.2 对比kafka
