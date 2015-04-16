#安装相应的jar包到maven仓库

##安装2个项目

有2个项目的jar包需要安装，分别是：

-	search-sqlparams-1.3.0.jar   主要用于查询参数的解析
-	search-core-1.3.0.jar  整体的查询流程体系，需要使用上述参数解析包

以search-sqlparams-1.3.0.jar为例来说下安装步骤：

-	第一步：fork search-sqlparams 项目（这两个项目都是maven环境），把项目源代码拉取到本地
-	第二步：在该项目的目录下，执行 **mvn install** 命令，就会将该项目编译并且安装jar包到maven仓库中
-	第三步：在该项目的目录下，执行 **mvn source:jar install** 命令，就会将该项目的源码包安装到maven仓库中

对于search-core项目同理，执行上述步骤，search-core一定要在search-sqlparams之后执行

这样操作，主要是方便自己去查看和修改源码

##直接获取项目的jar包

如果不想关心项目的源码，想直接获取jar包和源码包，下载地址如下：

[search-sqlparams-1.3.0.jar和search-core-1.3.0.jar](https://git.oschina.net/pingpangkuangmo/blog/tree/master/2015-4/%E9%99%84%E4%BB%B6)

#搭建查询demo

下载查询demo工程，使用ide导入demo工程。

-	第一步：准备数据库和数据

	demo项目中的类路径下，有一个search.sql文件，连上mysql数据库执行该sql文件，或者直接拷贝文件里面的内容，到mysql客户端直接执行这些内容。会创建一个search数据库，里面有a、b、c、d四张表和对应的数据

-	第二步：修改项目中jdbc.properties的配置文件，改成你实际的配置

-	第三步：在tomcat中运行该项目(部署路径设置为/)，访问http://localhost:8080/ 如下 **hello world** 形式，则表示项目搭建成功：![search-demo首页][1]

如果直接使用的是jar包，没有安装到maven仓库，则需要去掉demo工程中的以下依赖

	<!-- search -->
	<dependency>
		<groupId>com.dboper</groupId>
  		<artifactId>search-core</artifactId>
  		<version>1.3.0</version>
	</dependency>

然后将jar包直接放置到demo工程的classpath路径下

#多样化的实体查询API

##功能介绍

对实体提供多样化的查询API，是有一定的使用场景的，它适用于那些简单的查询实体间父子关系的场景。

主要功能：

-	1 对select语句查询出的平铺的结果展示进行聚合，使其具有父子关系

-	2 对select语句查询出的结果进行格式化，如true、false改为是、否

-	3 对于查询条件可以随意增添，支持类似mongodb的and or


##数据库数据说明

目前数据库中有4张实体表，分别是a、b、c、d，他们的关系是：

-	a:b=1:N
-	b:c=1:N
-	c:d=1:N

字段内容分别如下：

	a表： id、name、create_time
	b表： id、name、create_time、aId
	c表： id、name、create_time、bId
	d表： id、name、create_time、cId

##查询案例

分成2部分来说，查询实体间的关系和查询条件，以下请求都是post请求，请求体为json形式的字符串，同时content-type类型为application/json。返回的数据都是list类型的，所以下面就只显示一个数据的格式

###查询实体间的关系

-	1	查询b实体的内容

		{
			"entityColumns":["b"]
		}

	查询返回的数据格式如下：

	    {
	        "id": 1,
	        "name": "b1a1",
	        "createTime": 1427817600000,
	        "aId": "1"
	    }

-	2	查询b实体的内容，同时想知道其父实体a的内容

		{
			"entityColumns":["b","a@mapa"],
			"tablesPath":"b left join a"
		}

	查询返回的数据格式如下，b实体中有一个a属性，属性值为a实体的内容:
	
	    {
	        "id": 1,
	        "createTime": 1427817600000,
	        "aId": "1",
	        "a": {
	            "id": 1,
	            "createTime": 1427817600000,
	            "name": "a1"
	        },
	        "name": "b1a1"
	    }
	
	其中a@mapa分成三部分a @map a，第一部分返回格式中的属性名称,第二部分表示该属性是一个map形式，第三部分表示实体，综合起来就是实体a要作为实体b的一个属性，并且属性名称为a。
	
	其中tablesPath也可以不写，使用默认配置，但是b left join a和 b join a的结果是不一样的，所以用户可以根据需求来自行配置。支持join 、left join 、right join


-	3	查询b实体的内容，同时想知道它所包含的所有c实体

		{
			"entityColumns":["b","cs@listc"],
			"tablesPath":"b left join c"
		}

	查询返回的数据格式如下，b实体中有一个cs属性，该属性是c实体的一个集合

		{
	        "id": 1,
	        "createTime": 1427817600000,
	        "aId": "1",
	        "name": "b1a1",
	        "cs": [
	            {
	                "id": 1,
	                "createTime": 1428076800000,
	                "name": "c1b1a1",
	                "bId": "1"
	            },
	            {
	                "id": 2,
	                "createTime": 1428076800000,
	                "name": "c2b1a1",
	                "bId": "1"
	            }
	        ]
	    }

	其中cs@listc分成三部分 cs @list c，第一部分cs表示返回数据格式中的属性，第二部分表示该属性是一个集合形式,第三部分表示c实体，综合起来就是c实体要作为b实体cs属性的一个成员。

	tablesPath同上。

-	4	查询b实体的内容，同时想知道它的父实体a和它所包含的所有的c实体的内容

		{
			"entityColumns":["b","a@mapa","cs@listc"],
			"tablesPath":"b left join a left join c"
		}

	返回的数据格式如下，b实体有一个a属性，属性类型为map，值为a实体的内容；b实体有一个cs属性，属性类型为list，值为c实体的集合
		
		{
	        "id": 1,
	        "createTime": 1427817600000,
	        "aId": "1",
	        "a": {
	            "id": 1,
	            "createTime": 1427817600000,
	            "name": "a1"
	        },
	        "name": "b1a1",
	        "cs": [
	            {
	                "id": 1,
	                "createTime": 1428076800000,
	                "name": "c1b1a1",
	                "bId": "1"
	            },
	            {
	                "id": 2,
	                "createTime": 1428076800000,
	                "name": "c2b1a1",
	                "bId": "1"
	            }
	        ]
	    }

	其中tablesPath可以随意写，只要真实逻辑正确，不在乎顺序，如
	
		a right join b left join c
		b left join c left join a

		c right join a right join b 就不行，不是正常逻辑，
		因为 c right join a 没有直接的关系（即使有的情况是可以查出来的，有的情况也是查不出来的，最好不要这样使用）

-	5	查询b实体的内容，同时想知道它的父实体a和它所包含的所有的c实体的内容，以及它所包含的所有d实体的内容

		{
			"entityColumns":["b","a@mapa","cs@listc","ds@listd"],
			"tablesPath":"b left join a left join c left join d"
		}
		
	同理，在上一个格式的基础上，添加了一个ds的属性，该属性是d实体的集合，下面数据太长就省去了集合中的一部分实体

		{
	        "id": 1,
	        "createTime": 1427817600000,
	        "aId": "1",
	        "a": {
	            "id": 1,
	            "createTime": 1427817600000,
	            "name": "a1"
	        },
	        "name": "b1a1",
	        "ds": [
	            {
	                "id": 4,
	                "createTime": 1428768000000,
	                "name": "d2c2b1a1",
	                "cId": "2"
	            }
	        ],
	        "cs": [
	            {
	                "id": 2,
	                "createTime": 1428076800000,
	                "name": "c2b1a1",
	                "bId": "1"
	            }
	        ]
	    }

###查询条件

上面说完了查询实体间的关系，现在来看看在上面的基础上如何添加查询条件。上一部分内容就是整个查询体系search-core项目所做的事情（还有很多其他事情，之后再详谈），对于查询条件部分则是search-sqlparams项目的主要功能，可以参见[cms查询系统（二）json形式参数的设计与解析](http://my.oschina.net/pingpangkuangmo/blog/393405)

-	1	最简单的查询条件

		{
			"entityColumns":["b"],
			"params":{
				"b.id":1
			}
		}

	查询条件为b.id=1。下面的查询就只写params中的内容

-	2	一般查询

		{
			"b.id@>":1
		}

	查询条件为b.id>1

		同理，支持的查询条件还有如下：
		@=  @!=  @>  @>=  @<  @<=  @is

-	3	in not in 查询

		{
			"b.id@in":[1,2,3]
		}

		{
			"b.id@notIn":[1,2,3]
		}

-	4	时间查询

		{
			"b.create_time@time>":"2014-4-3"
		}

		{
			"b.create_time@full_time>":"2014-4-3 12:23:23"
		}
		
		同理还支持的查询操作为：
		@time> @time>= @time< @time<=
		@full_time> @full_time>= @full_time< @full_time<=
		还可以支持自定义扩展

-	5	like 查询

		{
			"b.name@like":"%a1%"
		}

	其中%用法和数据库保持一致

-	6	and 查询

		{
			"entityColumns":["b"],
  			"tablesPath":"b join a",
			"params":{
              	"b.name@like":"%a%",
				"a.id":1
			}
		}

	表示要查询的条件为b.name like a 同时a.id=1。

	这时候tablesPath为b和a实体间的关系，params中的查询条件就可以随意的指定a、b中要查询的字段。

	查询条件之间默认是and的关系

-	7	or 查询

		{
			"$or":{
				"b.id@<2"，
				"b.id@>3"
			}
		}
	上述表示的查询条件为 b.id<2 or b.id>3

	查询条件之间如果想使用or的关系，则使用$or将他们包裹起来。

-	8	and or 混合查询

		{
			"$or":{
				"b.id@<2"，
				"$and":{
					"b.create_time@time>":"2014-4-3",
					"b.create_time@time<":"2014-4-7"
				}
			}，
			"b.name@like":"%a%"
		}

	上述表示的查询条件为：

	两个b.create_time条件构成and关系，然后再与b.id条件构成or的关系，再与b.name条件构成and关系

先有一个基本的了解与认识，之后再详细说明细节

[1]: http://static.oschina.net/uploads/space/2015/0415/074223_7CPc_2287728.png
