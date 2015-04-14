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

#搭建查询demo

下载查询demo工程，使用ide导入demo工程。

-	第一步：准备数据库和数据

	demo项目中的类路径下，有一个search.sql文件，连上mysql数据库执行该sql文件，或者直接拷贝文件里面的内容，到mysql客户端直接执行这些内容。会创建一个search数据库，里面有a、b、c、d四张表和对应的数据

-	第二步：修改项目中jdbc.properties的配置文件，改成你实际的配置

-	第三步：在tomcat中运行该项目(部署路径设置为/)，访问http://localhost:8080/ 如下**hello world**形式，则表示项目搭建成功：![search-demo首页][1]

#多样化的实体查询API

对实体提供多样化的查询API，是有一定的使用场景的，它适用于那些简单的查询实体间父子关系的场景。

主要功能：

-	1 对select语句查询出的平铺的结果展示进行聚合，使其具有父子关系

-	2 对select语句查询出的结果进行格式化，如true、false改为是、否

-	3 对于查询条件可以随意增添，支持类似mongodb的and or

##简单的查询



[1]: http://static.oschina.net/uploads/space/2015/0415/074223_7CPc_2287728.png
