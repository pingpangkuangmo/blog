#mysql 常见锁问题分析

#1 参考资料

-	[The InnoDB Transaction Mode and Locking-官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-model.html)
-	[MySQL 加锁处理分析](http://hedengcheng.com/?p=771#_Toc374698317)
-	[Innodb中的事务隔离级别和锁的关系](http://tech.meituan.com/innodb-lock.html)

#2 要明确的概念

-	不可重复读和幻读的区别
-	快照读和当前读
-	事务的隔离级别
-	record lock、gap lock、next-key lock

##2.1 不可重复读和幻读的区别

select数据的不变性可以细分成2部分

-	第一部分就是：对原有数据的不可修改性，如update delete，通过行锁锁住记录就可以实现不可修改
-	另一部分就是：对于新增数据的限制，这时候就不能通过行锁来解决了，这时候就需要通过gap lock来解决

如果仅仅满足了第一部分可以叫可重复读，如果也满足了第二部分就算是解决了幻读的问题。

而目前mysql的innodb数据库引擎实现的Repeatable reads不仅仅解决了上述的第一部分也解决第二部分，即Repeatable reads级别下已经解决了幻读问题。

##2.2 快照读和当前读

快照读： 如普通的select * from t where id>=6;采用MVCC（多版本并发控制）仅仅读取该事务号及其之前的数据

当前读：如select * from t where id>=6 for update；读取的是最新提交的事务号及其之前的数据

##2.3 事务的隔离级别

Read committed 的隔离级别：只能读到别人已提交的数据，未提交的数据读不到，RC的隔离级别存在不可重复读和幻读的现象，即在同一个事务内，第一次select查询出一定结果后，别的客户端此时又修改了源数据和提交了新的数据，第二次select是可以查出修改后的数据和新提交的数据的，这就导致了和第一次select的数据不一致的问题

Repeatable reads 的隔离级别：比起Read committed，解决了不可重复读的现象，而mysql的innodb数据库引擎实现的Repeatable reads也解决了幻读问题。

-	对于快照读中的幻读（即select * from t where id>=6出现的幻读）采用的解决方式是采用MVCC（多版本并发控制）
-	对于当前读中的幻读（即select * from t where id>=6 for update出现的幻读）采用的解决方式是gap lock

#3 问题分析

如下的一个事务并发执行

	start transaction;
	DELETE FROM t WHERE id =6;
	INSERT INTO t VALUES(6);
	commit;

就隔离级别和id唯一索引、id非唯一索引组合等情况展开分析以下内容：

-	使用了什么锁？阻塞情况？
-	是否会出现死锁？

##3.1 Read committed和唯一索引id

##3.2 Read committed和非唯一索引id

##3.3 Repeatable reads和唯一索引id

创建表的sql:

	create table m (
		id int ,
		primary key (id)
	);

填充数据 1、2、6、8

	insert into m values(1),(2),(6),(8);

步骤1：客户端A

	start transaction;
	delete from m where id =6;

步骤2：客户端B
	
	start transaction;
	delete from m where id =6;

步骤3：客户端A
	
	insert into m value(6);

步骤4：客户端B
	
	insert into m value(6);


###3.3.1 删除一个已存在的值

其中delete from m where id =6语句会对索引中id=6的记录加上next-key lock,但是由于where id=6的查询条件结果是确定的，即不会出现幻读的情况，所以仅仅对id=6的记录加上一个record lock即可，即由next-key lock降级到了record lock。

所以当客户端A执行完毕步骤1后，客户端B执行步骤2的时候，由于已经存在了record lock，所以客户端B会被阻塞，等待客户端A的record lock的释放，现象如下：

![输入图片说明](https://static.oschina.net/uploads/img/201602/28174802_CoW2.png "在这里输入图片标题")

###3.3.2 删除一个不存在的值

上述的id=6全部换成id=5,即客户端A执行delete from m where id=5;由于记录不存在，所以只会在索引(2,6)区间中加上gap lock。此时如果客户端B也同样执行delete from m where id=5，由于记录不存在，也只会在索引(2,6)区间中加上gap lock，这两个gap lock之间不冲突，可以同时存在。

此时客户端执行insert into m value(5),因为insert语句会添加一个 insert intention gap lock（见官方文档[insert intention gap lock](https://dev.mysql.com/doc/refman/5.7/en/innodb-record-level-locks.html)），其中这个锁和gap lock是可以冲突的，此时插入的数值5刚好在上述客户端B创建的gap lock锁定的区间中，所以此时客户端A是要等待客户端B释放gap lock的，即被阻塞了

此时客户端B同样执行insert into m value(5)，也会因为客户端A创建的gap lock而造成阻塞，此时客户端A、B相互阻塞造成死锁，现象如下

![输入图片说明](https://static.oschina.net/uploads/img/201602/28175631_rXWu.png "在这里输入图片标题")

发生死锁后，innode引擎自动检测到死锁，会让一个进行释放，另一个得到执行

##3.4 Repeatable reads和非唯一索引id

创建表的sql:

	create table t (
		id int ,
		key (id)
	);

填充数据 1、2、6、8

	insert into t values(1),(2),(6),(8);

步骤1：客户端A

	start transaction;
	delete from t where id =6;

步骤2：客户端B
	
	start transaction;
	delete from t where id =6;

步骤3：客户端A
	
	insert into t value(6);

步骤4：客户端B
	
	insert into t value(6);

###3.4.1 删除一个已存在的值

其中delete from t where id =6语句会对索引中id=6的记录加上next-key lock，即id=6的记录本身加上record lock，同时(2,6)、(6,8) 这两个区间会加上gap lock。

所以当客户端A执行完毕步骤1后，客户端B执行步骤2的时候，由于已经存在了record lock，所以客户端B会被阻塞，等待record lock的释放，现象如下：

![客户端B被阻塞](https://static.oschina.net/uploads/img/201602/27223803_kG5N.png "客户端B被阻塞")

###3.4.2 删除一个不存在的值

假如上述的id=6全部换成id=5，执行 delete from t where id =5的话，即delete 一个不存在的值，则只会对索引的(2,6)区间加上gap lock。客户端B就不会阻塞，同样的在索引(2,6)区间加上gap lock，gap lock之间不冲突的，即这时的客户端B不会阻塞。

这时客户端A执行 insert into t value(5)会阻塞，因为insert语句会添加一个 insert intention gap lock（见官方文档[insert intention gap lock](https://dev.mysql.com/doc/refman/5.7/en/innodb-record-level-locks.html)），其中这个锁和gap lock是可以冲突的，此时插入的数值5刚好在上述客户端B创建的gap lock锁定的区间中，所以此时客户端A是要等待客户端B释放gap lock的，即被阻塞了

![输入图片说明](https://static.oschina.net/uploads/img/201602/27230922_oK1S.png "在这里输入图片标题")

而客户端A执行的insert into t value(5)所加入的insert intention gap lock是不会和自己创建的gap lock冲突的，即如果没有其他gap lock的话，客户端A往自己创建的gap lock区间中insert值是不被阻塞的

![输入图片说明](https://static.oschina.net/uploads/img/201602/27231211_dvkd.png "在这里输入图片标题")

假如此时客户端B同样执行insert into t value(5)操作，则会因为客户端A创建的gap lock而等待，此时客户端A B在相互等待对方释放gap lock，造成死锁，现象如下：

![输入图片说明](https://static.oschina.net/uploads/img/201602/27232746_pY3t.png "在这里输入图片标题")

发生死锁后，innode引擎自动检测到死锁，会让一个进行释放，另一个得到执行

#4 案例演示当前读和快照读

创建表的sql:

	create table t (
		id int ,
		key (id)
	);

填充数据 1、2、6、8

	insert into t values(1),(2),(6),(8);

步骤1：客户端A

	start transaction;
	select * from t where id>=6;

步骤2：客户端B

	start transaction;
	insert into t value(10);
	commit;

步骤3：客户端A

	start transaction;
	select * from t where id>=6;
	select * from t where id>=6 for update;


步骤3中是第一个select是看不到客户端B新增的数据的，因为他是快照读，读取的是该事务号及其之前的数据

步骤3中的第二个select是可以看到客户端B新增的数据的，因为它是当前读，读取的是最新提交的事务号及其之前的数据

现象如下：

![输入图片说明](https://static.oschina.net/uploads/img/201602/29112851_Wsnd.png "在这里输入图片标题")

同时如下的更新语句也是当前读

	update t set key=2 where id>3


