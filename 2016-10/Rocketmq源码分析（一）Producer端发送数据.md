# 1 系列

-	[整体架构图](https://my.oschina.net/pingpangkuangmo/blog/753742)
-	producer端发送消息
-	broker端接收消息
-	broker端消息的存储
-	consumer消费消息
-	分布式事务的实现
-	定时消息的实现
-	关于顺序消费
-	关于重复消息
-	关于高可用

# 2 Producer的整体架构


# 3 Producer发送消息的过程

# 4 Producer要关注的问题

-	更新基础信息

-	怎么管理与多个broker之间的连接，是否共享

-	流量过大、broker连接不上等情况消息怎么处理

-	消息的顺序发送

-	heartbeat

-	发送超时

-	同步还是异步

-	发送结果

