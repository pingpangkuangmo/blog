#sql 常见锁问题分析

#1 参考资料

-	[The InnoDB Transaction Model and Locking](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-model.html)
-	[MySQL 加锁处理分析](http://hedengcheng.com/?p=771#_Toc374698317)

#2 要明确的概念

-	事务的隔离级别
-	record lock、gap lock、next-key lock

#2 问题分析

如下的一个事务并发执行

	start transaction;
	DELETE FROM t WHERE id =6;
	INSERT INTO t VALUES(6);
	commit;

就隔离级别和id唯一索引、id非唯一索引组合等情况展开分析是否会出现死锁

##2.1 RC和唯一索引id

##2.2 RC和非唯一索引id

##2.3 RR和唯一索引id

##2.4 RR和非唯一索引id