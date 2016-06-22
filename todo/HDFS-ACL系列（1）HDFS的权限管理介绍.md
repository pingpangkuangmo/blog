# 1 目录

# 2 HDFS的权限管理介绍

HDFS的权限管理分成2大部分：

-	类似linux的基本权限管理（粗粒度）

	针对管理对象分三种：user、group、other方式的权限管理方式

	-	user:即目录或文件的owner
	-	group:即上述owner所在的组
	-	other：其他用户的统称

-	ACL方式的权限管理（细粒度）

	可以精确控制到某个user、某个group具有对应的权限

2种方式具体见下图

![基本权限方式和ACL权限方式](https://static.oschina.net/uploads/img/201606/21224904_KZpt.png "基本权限方式和ACL权限方式")

基本的权限方式如同linux中目录或文件的权限管理方式，它是一种粗粒度的，不能控制到某一个用户如t1。ACL的权限管理方式就可以，如下

user:t1:r-x

表示用户t1具有读和可执行的权限（在HDFS中没有可执行的概念，暂且这么叫吧，这里的x权限就是进入目录和列出目录下的文件或内容的权限）

# 3 HDFS基本权限管理

创建一个文件或目录，初始的基本权限是多少呢？经历如下2过程：

## 3.1 默认的文件或目录权限

创建文件或目录的时候可以指定权限，没有指定的话，就使用默认的权限，如下

目录：777

文件：666

## 3.2 应用客户端配置的uMask

注意：这个配置是客户端可以配置的，即客户端自己主宰创建文件或目录的权限。

umask的作用：

就是对HDFS基本权限的限制，如022分别表示：0表示对owner没有限制，2表示对group不允许有写权限，2表示对other不允许有写权限。

应用客户端配置的umask其实就是拿用户设置的权限减去上述umask权限，就得到文件或目录的最终权限了。如创建目录，默认777权限，然后减去umask权限022就等于755，即默认情况下owner拥有读写和可执行权限，owner所在group拥有读和可执行权限，other拥有读和可执行的权限

下面重点来说下umask的获取过程：

-	首先尝试获取配置文件中的fs.permissions.umask-mode属性，如022，然后构建出umask权限对象

-	如果上述属性没有配置的话，获取dfs.umask属性（过时了，推荐用第一种方式），这种方式的umask是10进制形式，如上述的022=2x16+2=18即这里的umask要配置成18

-	如果上述属性也没有配置的话，默认使用18即022来构建出umask对象

至此，在没有ACL设置的情况下创建文件或目录的权限即上述过程了。

## 3.3 Sticky Bit

# 4 HDFS ACL权限管理

## 4.1 ACL权限管理模型

AclEntry对象，如

user:t1:r-x、group:g1:rwx、other::--x、mask::r-x

default:user:t1:r-x、default:group:g1:r-x、default:other::r-x、default:mask::r-x

包含4部分内容：

	public class AclEntry {
	  private final AclEntryType type;
	  private final String name;
	  private final FsAction permission;
	  private final AclEntryScope scope;
    }

-	name:即名称，如上述的t1、g1，也可以为null如上述的mask::r-x

-	FsAction：权限信息，总共有如下权限

		NONE("---"),
		EXECUTE("--x"),
		WRITE("-w-"),
		WRITE_EXECUTE("-wx"),
		READ("r--"),
		READ_EXECUTE("r-x"),
		READ_WRITE("rw-"),
		ALL("rwx");

-	AclEntryScope:枚举值，值为ACCESS、DEFAULT

	ACCESS：表示该条ACL是针对本目录或文件的，前缀省略不写

	DEFAULT：只存在于目录中，前缀是default:，一旦在该目录中创建子目录或文件，子目录或文件的ACL就会继承该条ACL权限。即如果父目录中含有一个default:user:t1:r-x记录，则在创建子目录或者文件的时候，都会自动加上一条user:t1:r-x的记录

-	AclEntryType:枚举值，值为USER、GROUP、OTHER、MASK

	USER:如上述的user:t1:r-x表示t1是一个用户

	GROUP：如上述的group:g1:rwx表示g1是一个组
	
	OTHER：如上述的other::--x表示剩余的所有用户的统称

	MASK：如上述的mask::r-x，主要用于过滤，如上述的group:g1:rwx，虽然这里是rwx权限，但是真正在判定权限的时候，还是要与mask::r-x进行and操作，即过滤了w权限，所以group:g1:rwx的实际权限是group:g1:r-x，如下图所示

	![mask的权限过滤](https://static.oschina.net/uploads/img/201606/22075459_ghE8.png "mask的权限过滤")

	其实mask只作用在name值不为null的user和所有的group上，这个后面再详细说明

## 4.2 ACCESS scope中的mask

### 4.2.1 mask值的主动修改

ACCESS scope中的mask对应的权限就是HDFS基本权限中的group权限。如下所示：

![mask对应group权限](https://static.oschina.net/uploads/img/201606/22080307_5Hrm.png "mask对应group权限")

所以如下2种方式都可以更改基本权限中的group权限：

-	hdfs dfs -chmod 751

	如修改基本权限为751，则mask取group权限则为5，即r-x权限，如下图所示

	![chmod修改mask权限](https://static.oschina.net/uploads/img/201606/22080803_tgDK.png "chmod修改mask权限")

-	直接修改mask acl中的权限值

	如hdfs dfs -setfacl -m mask::--x /user/lg/acl，修改了mask值，同时也修改了基本权限中的group权限

	![修改mask影响group](https://static.oschina.net/uploads/img/201606/22081116_pR0J.png "修改mask影响group")

### 4.2.2 mask值的被动重新计算

当acl新添加或者删除的时候，都会触发mask的重新计算，计算方式就是：

所有name不为null的user类型的acl和所有的group类型的acl的权限取并集

例子如下：

![mask的重新计算](https://static.oschina.net/uploads/img/201606/22083739_l4u7.png "mask的重新计算")

这时候基本权限中的group权限也会随着mask的改变而改变。

目前到这里总结下mask:

主要设计成对其他user和group类型的权限过滤，你主动去设置mask，此时会起到一定的过滤作用，

但是一旦重新添加或者删除acl的时候，mask的值就被重新计算了，也就是说你之前设置的没啥用了，这个就意味着mask没啥鸟用，当你没注意执行chmod修改权限，就会造成别人如t1用户突然少了某些权限，一旦新增一个t2用户的acl权限，t1用户的权限又突然回来了，这个也算是个坑吧，还是我没彻底搞明白，欢迎来指正和讨论。

## 4.3 DEFAULT scope中的mask

## 4.4 权限验证过程