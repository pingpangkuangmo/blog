#1 系列目录

-	ZooKeeper源码环境搭建


#2 搭建步骤

##2.1 到github中fork该项目
	
项目地址 [https://github.com/apache/zookeeper](https://github.com/apache/zookeeper)。fork完成之后就存至自己的仓库中了。

##2.2 clone上述自己的仓库地址到本地

先看下大体的代码格式：

![ZooKeeper的源码目录](https://static.oschina.net/uploads/img/201507/28080738_fgxz.png "ZooKeeper的源码目录")

##2.3 使用ant对源码编译成eclipse工程

首先选定一个分支，我自己选择branch-3.4分支来进行源码研究。即

	git checkout branch-3.4

上述源码还不是eclipse工程。需要使用ant eclipse命令来转换成eclipse工程。ant就不用再说了，自行网上搜索与配置。

	ant eclipse

这里来重点说说ant eclipse执行失败的问题。

-	1 上述命令会下载ant-eclipse-1.0.bin.tar.bz2文件，老是下载不成功，无法继续下去

	修改源码中build.xml中的配置，将地址

		<get src="http://downloads.sourceforge.net/project/ant-eclipse/ant-eclipse/1.0/ant-eclipse-1.0.bin.tar.bz2"

	更换成如下地址

		<get src="http://ufpr.dl.sourceforge.net/project/ant-eclipse/ant-eclipse/1.0/ant-eclipse-1.0.bin.tar.bz2"

-	2 还发现缺少依赖包 commons-collections

	在ivy.xml文件中加入如下配置
	
		<dependency org="commons-collections" name="commons-collections" rev="3.0"/>
	
上述两个问题解决后，再重新执行ant eclipse命令。

##2.4 导入项目到eclipse工程中

