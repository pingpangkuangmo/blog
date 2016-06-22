# 1 HDFS的权限管理介绍

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

# 2 HDFS基本权限管理

创建一个文件或目录，初始的基本权限是多少呢？经历如下2过程：

## 2.1 默认的文件或目录权限

创建文件或目录的时候可以指定权限，没有指定的话，就使用默认的权限，如下

目录：777

文件：666

## 2.2 应用客户端配置的uMask

注意：这个配置是客户端可以配置的，即客户端自己主宰创建文件或目录的权限。

umask的作用：

就是对HDFS基本权限的限制，如022分别表示：0表示对owner没有限制，2表示对group不允许有写权限，2表示对other不允许有写权限。

应用客户端配置的umask其实就是拿用户设置的权限减去上述umask权限，就得到文件或目录的最终权限了。如创建目录，默认777权限，然后减去umask权限022就等于755，即默认情况下owner拥有读写和可执行权限，owner所在group拥有读和可执行权限，other拥有读和可执行的权限

下面重点来说下umask的获取过程：

-	首先尝试获取配置文件中的fs.permissions.umask-mode属性，如022，然后构建出umask权限对象

-	如果上述属性没有配置的话，获取dfs.umask属性（过时了，推荐用第一种方式），这种方式的umask是10进制形式，如上述的022=2x8+2=18即这里的umask要配置成18

-	如果上述属性也没有配置的话，默认使用18即022来构建出umask对象

至此，在没有ACL设置的情况下创建文件或目录的权限即上述过程了。

# 3 HDFS ACL权限管理

## 3.1 ACL权限管理模型

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

基本权限也可以转化成ACL权限，例子如基本权限如drwxr-xr-x，对应的ACL权限如

user::rwx   对应user权限

（group::r-x）对应group权限

mask::r-x   对应group权限

other::r-x  对应other权限

上述修改基本权限或者修改对应的ACL权限都会相互影响。有一个例外：上述的group::r-x在创建的时候依据基本权限中的group权限，但是直接修改ACL中的group::r-x权限，并不直接影响基本权限中的group权限，这也是比较坑的地方，即group::r-x权限被造出来之后就像被遗弃了。

## 3.2 ACCESS scope中的mask

### 3.2.1 mask值的主动修改

ACCESS scope中的mask对应的权限就是HDFS基本权限中的group权限。如下所示：

![mask对应group权限](https://static.oschina.net/uploads/img/201606/22080307_5Hrm.png "mask对应group权限")

所以如下2种方式都可以更改基本权限中的group权限：

-	hdfs dfs -chmod 751

	如修改基本权限为751，则mask取group权限则为5，即r-x权限，如下图所示

	![chmod修改mask权限](https://static.oschina.net/uploads/img/201606/22080803_tgDK.png "chmod修改mask权限")

-	直接修改mask acl中的权限值

	如hdfs dfs -setfacl -m mask::--x /user/lg/acl，修改了mask值，同时也修改了基本权限中的group权限

	![修改mask影响group](https://static.oschina.net/uploads/img/201606/22081116_pR0J.png "修改mask影响group")

### 3.2.2 mask值的被动重新计算

当acl新添加或者删除的时候，都会触发mask的重新计算，计算方式就是：

所有name不为null的user类型的acl和所有的group类型的acl的权限取并集

例子如下：

![mask的重新计算](https://static.oschina.net/uploads/img/201606/22083739_l4u7.png "mask的重新计算")

这时候基本权限中的group权限也会随着mask的改变而改变。

目前到这里总结下mask:

主要设计成对其他user和group类型的权限过滤，你主动去设置mask，此时会起到一定的过滤作用，

但是一旦重新添加或者删除acl的时候，mask的值就被重新计算了，也就是说你之前设置的没啥用了，这个就意味着mask没啥鸟用，当你无意间执行chmod修改权限，就可能会造成别人如t1用户突然少了某些权限，一旦新增一个t2用户的acl权限，t1用户的权限又突然回来了，这个也算是个坑吧，不知道为啥这样设计，欢迎来讨论。

## 3.3 DEFAULT scope中的mask

DEFAULT scope中的mask和ACCESS scope中的mask基本类似，都是用于过滤权限。

### 3.3.1 mask值主动修改

上述ACCESS scope中mask值有2种方式，而DEFAULT scope中只有一种方式，就是

直接修改mask acl中的权限值

如hdfs dfs -setfacl -m default:mask::--x /user/lg/acl

![修改default中的mask](https://static.oschina.net/uploads/img/201606/22212422_AI1r.png "修改default中的mask")

修改之后上述default:user:t1:r-x 的实际权限就只是 --x了，即与default mask执行and操作的结果

### 3.3.2 mask值被重新计算

当acl新添加或者删除的时候，都会触发mask的重新计算，计算方式就是：

所有name不为null的user类型的acl和所有的group类型的acl的权限取并集。这里不再说明了，见ACCESS scope中的mask重新计算。

## 3.4 过程分析

前面介绍的是一些基本概念和理论，现在来看看下面几个过程具体会发生什么操作

### 3.4.1 权限验证过程

例子： 验证用户u1是否对路径/user/lg/acl有读权限

-	1 检查u1是否是该路径的owner
	
	如果是owner,则直接使用基本权限中的user权限来判定，代码如下

	![owner权限判定](https://static.oschina.net/uploads/img/201606/22214809_B0uC.png "owner权限判定")

-	2 遍历该路径的所有ACCESS scope的ACL权限

	如果当前ACL类型是user，并且用户名匹配，如user:u1:r-x，在判定时还需要将该r-x权限和基本权限中的group权限（即mask权限）执行and操作来作为实际的权限，如果此时mask::--x，执行and操作之后实际权限变成了--x权限，即仍然没有读权限

	如果当前ACL类型是group,并且该u1用户所属的groups中包含该group，同上述一样，仍然需要将该条ACL权限与基本权限中的group权限（即mask权限）执行and操作来判定最终的权限

	代码如下：

	![user和group acl的判定](https://static.oschina.net/uploads/img/201606/22214956_yebU.png "user和group acl的判定")

-	3 一旦该用户在上述权限中都没匹配到，则使用基本权限中的other权限来判定

	注意这里的没有匹配到的含义：如果有user:u1:--x权限，即匹配到了用户，但是该ACL并没有读权限，此时并不会去执行这部分的other权限

	![other的权限判定](https://static.oschina.net/uploads/img/201606/22215102_Oq0m.png "other的权限判定")

-	4 以上还未能找到相关权限则判定该用户无权限

### 3.4.2 创建文件或目录

-	1 如果创建文件或者目录时没有指定基本权限权限，则使用默认的基本权限权限

	目录：777

	文件：666

-	2 在上述基本权限的基础上应用客户端配置的umask权限，默认是022，则目录和文件分别变成
	
	目录：755

	文件：644

-	3 如果父目录中含有DEFAULT scope的ACL权限信息

	default:user::rwx和上述文件或目录基本权限中的user权限执行and操作作为最终的基本权限中的user权限
	default:mask::r-x和上述文件或目录基本权限中的group权限执行and操作作为最终的基本权限中的group权限
	default:other::--x和上述文件或目录基本权限中的other权限执行and操作作为最终的基本权限中的other权限

	例子如下：

	![基本权限的确定](https://static.oschina.net/uploads/img/201606/22220422_lt8A.png "基本权限的确定")

	default中的其他权限直接作为子目录或文件的ACCESS权限

	代码见：

	![父目录中的default权限转交给子文件或目录](https://static.oschina.net/uploads/img/201606/22220814_wFRi.png "父目录中的default权限转交给子文件或目录")

-	4 如果创建的是子目录，则将全部的default权限复制给子目录作为子目录的default权限

	
-	5 然后重新计算子文件或目录的基本权限（因为上述过程可能会修改基本权限）

	代码见

	![上述4和5过程](https://static.oschina.net/uploads/img/201606/22221224_0qLZ.png "上述4和5过程")

### 3.4.3 修改ACL

新增或删除或修改ACL操作的时候都会重新计算mask的值

ACCESS scope中的ACL变动会重新计算ACCESS scope中的mask

DEFAULT scope中的ACL变动会重新计算DEFAULT scope中的mask

mask的计算方式上面就说明了。