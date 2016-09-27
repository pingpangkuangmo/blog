准备将Netty的源码过一下，一来对自己是个总结消化的过程，二来希望对那些打算看Netty源码的人（已经熟悉Netty基本原理）能有一些帮助。目前所看Netty版本是4.1.3.Final。

# 1 目录

-	[Netty概览](http://my.oschina.net/pingpangkuangmo/blog/734051)
-	[EventLoopGroup分析](http://my.oschina.net/pingpangkuangmo/blog/742929)


# 2 概述关系

Channel、ChannelHandler、ChannelPipeline三者的关系

-	1 每一个Channel都有一个ChannelPipeline，ChannelPipeline主要掌管着该Channel的事件的处理
-	2 ChannelPipeline可以有多个ChannelHandler，这些ChannelHandler构建成一个双向链表，共同来处理着上述Channel的事件

# 3 Channel接口定义

先来简单看下接口实现情况

![Channel接口实现情况](https://static.oschina.net/uploads/img/201609/16092132_VE04.png "Channel接口实现情况")

-	1 Channel：Netty对于Channel接口的功能定义
-	2 AbstractChannel：对于上述接口的抽象实现
-	3 AbstractNioChannel、AbstractEpollChannel：分别实现poll、epoll模型下的Channel
-	4 AbstractNioByteChannel、NioSocketChannel：对于poll模型下的客户端Channel的实现
-	5 AbstractNioMessageChannel、NioServerSocketChannel：对于poll模型下的服务器端Channel的实现

## 3.1 Channel功能定义

它提供了如下信息：

1 该Channel的状态信息

	boolean isOpen();
	boolean isRegistered();
	boolean isActive();

这里的isOpen是指：jdk底层的Channel一旦new出来就是open=true的状态，一旦调用了close方法open=false，所以这里的open更像是是否关闭的概念

这里的isRegistered是指：netty的Channel是否已经注册到了EventLoop中


# 4 ChannelHandler接口定义

# 5 ChannelPipeline接口定义

# 6 后续

下一篇就要详细描述下NioEventLoop对于IO事件的处理，即ChannelPipeline的处理流程。
