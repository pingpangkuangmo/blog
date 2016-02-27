#sql 常见锁问题分析

#1 参考资料

-	[The InnoDB Transaction Mode and Locking-官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-model.html)
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

就隔离级别和id唯一索引、id非唯一索引组合等情况展开分析以下内容：

-	使用了什么锁？阻塞情况？
-	是否会出现死锁？

##2.1 RC和唯一索引id

##2.2 RC和非唯一索引id

##2.3 RR和唯一索引id

创建表的sql:

	create table m (
		id int ,
		primary key (id)
	);

填充数据 1、2、6、8

	insert into m values(1),(2),(6),(8);

开启客户端A：

<table>
<thead>
<tr>
  <th></th>
  <th>客户端A</th>
  <th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
  <td>步骤1</td>
  <td>
		start transaction;
		delete from t where id =6;
  </td>
  <td>
  </td>
</tr>
<tr>
  <td>步骤2</td>
  <td></td>
  <td>
		start transaction;
		delete from t where id =6;
  </td>
</tr>
<tr>
  <td>步骤3</td>
  <td>
		insert into t value(6);
  </td>
  <td>
  </td>
</tr>
<tr>
  <td>步骤4</td>
  <td>
  </td>
  <td>
		insert into t value(6);
  </td>
</tr>
</tbody>
</table>

###2.3.1 删除一个已存在的值

其中delete from t where id =6语句会对索引中id=6的记录加上next-key lock,但是由于where id=6的查询条件结果是确定的，即不会出现幻读的情况，所以仅仅对id=6的记录加上一个record lock即可，即由next-key lock降级到了record lock。

所以当客户端A执行完毕步骤1后，客户端B执行步骤2的时候，由于已经存在了record lock，所以客户端B会被阻塞，等待客户端A的record lock的释放，现象如下：



###2.3.2 删除一个不存在的值

##2.4 RR和非唯一索引id

创建表的sql:

	create table t (
		id int ,
		key (id)
	);

填充数据 1、2、6、8

	insert into t values(1),(2),(6),(8);

开启客户端A：

<table>
<thead>
<tr>
  <th></th>
  <th>客户端A</th>
  <th>客户端B</th>
</tr>
</thead>
<tbody>
<tr>
  <td>步骤1</td>
  <td>
		start transaction;
		delete from t where id =6;
  </td>
  <td>
  </td>
</tr>
<tr>
  <td>步骤2</td>
  <td></td>
  <td>
		start transaction;
		delete from t where id =6;
  </td>
</tr>
<tr>
  <td>步骤3</td>
  <td>
		insert into t value(6);
  </td>
  <td>
  </td>
</tr>
<tr>
  <td>步骤4</td>
  <td>
  </td>
  <td>
		insert into t value(6);
  </td>
</tr>
</tbody>
</table>

###2.4.1 删除一个已存在的值

其中delete from t where id =6语句会对索引中id=6的记录加上next-key lock，即id=6的记录本身加上record lock，同时(2,6)、(6,8) 这两个区间会加上gap lock。

所以当客户端A执行完毕步骤1后，客户端B执行步骤2的时候，由于已经存在了record lock，所以客户端B会被阻塞，等待record lock的释放，现象如下：

![客户端B被阻塞](https://static.oschina.net/uploads/img/201602/27223803_kG5N.png "客户端B被阻塞")

###2.4.2 删除一个不存在的值

假如上述的id=6全部换成id=5，执行 delete from t where id =5的话，即delete 一个不存在的值，则只会对索引的(2,6)区间加上gap lock。客户端B就不会阻塞，同样的在索引(2,6)区间加上gap lock，gap lock之间不冲突的，即这时的客户端B不会阻塞。

这时客户端A执行 insert into t value(5)会阻塞，因为insert语句会添加一个 insert intention gap lock（见官方文档[insert intention gap lock](https://dev.mysql.com/doc/refman/5.7/en/innodb-record-level-locks.html)），其中这个锁和gap lock是可以冲突的，此时插入的数值5刚好在上述客户端B创建的gap lock锁定的区间中，所以此时客户端A是要等待客户端B释放gap lock的，即被阻塞了

![输入图片说明](https://static.oschina.net/uploads/img/201602/27230922_oK1S.png "在这里输入图片标题")

而客户端A执行的insert into t value(5)所加入的insert intention gap lock是不会和自己创建的gap lock冲突的，即如果没有其他gap lock的话，客户端A往自己创建的gap lock区间中insert值是不被阻塞的

![输入图片说明](https://static.oschina.net/uploads/img/201602/27231211_dvkd.png "在这里输入图片标题")

假如此时客户端B同样执行insert into t value(5)操作，则会因为客户端A创建的gap lock而等待，此时客户端A B在相互等待对方释放gap lock，造成死锁，现象如下：

![输入图片说明](https://static.oschina.net/uploads/img/201602/27232746_pY3t.png "在这里输入图片标题")

发生死锁后，innode引擎自动检测到死锁，会让一个进行释放，另一个得到执行




