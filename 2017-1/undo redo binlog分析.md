# 0 推荐阅读

-	[MySQL数据库InnoDB存储引擎Log漫游(1)](http://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=100000015&idx=1&sn=c82e5f09244399e835c0184ee5718fe5&chksm=683dcb5d5f4a424b7068639c8e6f69ed7fb17885dc3bc04bc11369db20ecee9a0c630b91be08&mpshare=1&scene=23&srcid=0113v2HODbNTio1EpT50yE2Y#rd)
-	[MySQL数据库InnoDB存储引擎Log漫游(2)](http://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=100000024&idx=1&sn=b1b518e8a0fab63b776227d35b211205&chksm=683dcb4a5f4a425cd1d4e32bbd126119e9123802625a64f553e1a15836b1cae05fefd94f4642&mpshare=1&scene=23&srcid=0113iNZSBODnOFJa5WAMPzOh#rd)
-	[MySQL数据库InnoDB存储引擎Log漫游(3)](http://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=100000026&idx=1&sn=f8ed552d3e0cd7d71e23be6de16f7d5e&chksm=683dcb485f4a425e81ce5365cf95965faca89a1bfe414566fa85b191d5c8c3dd0fe7ee526086&mpshare=1&scene=23&srcid=0113eUAFsl7h5gJegVWite35#rd)
-	[MySQL的CrashSafe和Binlog的关系](http://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247483833&idx=1&sn=d96f2994b9dc4d566bbc7ac5a92b3aad&chksm=e83dcbebdf4a42fd8ece0252dbfa46a65d8c5bca4297a7ca4c93ea60f596ea145629b9abedfc&mpshare=1&scene=23&srcid=0116ytTL2CbBVsNcdS48Tjon#rd)

# 1 问题总结

## 1.1 undo redo binlog的问题：

-	1 undo redo 都可以实现持久化，他们的流程是什么？为什么选用redo来做持久化？
-	2 undo、redo结合起来实现原子性和持久化，为什么undo log要先于redo log持久化？
-	3 undo为什么要依赖redo？
-	4 日志内容可以是物理日志，也可以是逻辑日志？他们各自的优点和缺点是？
-	5 redo log最终采用的是物理日志加逻辑日志，物理到page，page内逻辑。还存在什么问题？怎么解决？Double Write
-	6 undo log为什么不采用物理日志而采用逻辑日志？
-	7 为什么要引入Checkpoint？
-	8 引入Checkpoint后为了保证一致性需要阻塞用户操作一段时间，怎么解决这个问题？（这个问题还是很有普遍性的，redis、ZooKeeper都有类似的情况以及不同的应对策略）又有了同步Checkpoint和异步Checkpoint
-	9 开启binlog的情况下，事务内部2PC的一般过程（含有2次持久化，redo log和binlog的持久化）
-	10 解释上述过程，为什么binlog的持久化要在redo log之后，在存储引擎commit之前？
-	11 为什么要保持事务之间写入binlog和执行存储引擎commit操作的顺序性？（即先写入binlog日志的事务一定先commit）
-	12 为了保证上述顺序性，之前的办法是加锁prepare\_commit\_mutex，但是这极大的降低了事务的效率，怎么来实现binlog的group commit？
-	13 怎么将redo log的持久化也实现group commit？至此事务内部2PC的过程，2次持久化的操作都可以group commit了，极大提高了效率

## 1.2 锁

-	1 lock和latch
-	2 自增长与锁（表锁、互斥量）
-	3 Record Lock、gap Lock、next-key lock

# 2 问题解释

-	1 undo redo 都可以实现持久化，他们的流程是什么？为什么选用redo来做持久化？