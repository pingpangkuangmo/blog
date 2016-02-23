#1 gecko概述

最近在研究metaq消息队列，它里面用到的NIO通信框架是gecko，文档是这么描述的

	Gecko是一个Java NIO的通讯组件，它在一个轻量级的NIO框架的基础上提供了更高层次的封装和功能。

	支持的RPC调用方式包括RR(request-response)和pipeline。

-	0 可插拔的协议设计 
-	1 连接池 
-	2 分组管理和负载均衡 
-	3 Failover/Retry 
-	4 重连管理 
-	5 同步和异步调用

本文就按照日常NIO通信框架和RPC所面临的问题来看下gecko是怎么解决和实现的

#2 gecko实现NIO通信框架

##2.1 NIO类库的使用

像Netty、Mina这种NIO通信框架都是不使用Jdk自带的NIO类库，自己重写NIO类库。而gecko则是使用的是jdk自带的NIO类库，具体的差别，我现在还不太了解

##2.2 线程模型的选择

先简单描述下NIO形式下的线程模型，具体参考这篇文章[infoq:Netty系列之Netty线程模型](http://www.infoq.com/cn/articles/netty-threading-model)

-	传统BIO模型

![传统BIO模型](https://static.oschina.net/uploads/img/201510/20083738_I5mX.png "传统BIO模型")

比较简单，每来一个连接就创建一个线程来处理。这时候最大线程数就成了连接数的限制。

-	Reactor单线程模型

![Reactor单线程模型](https://static.oschina.net/uploads/img/201510/31082759_7QTr.png "Reactor单线程模型")

使用一个线程，使用Selector来监听连接的建立和连接的读写事件，然后仍然是该线程去完成事件的处理。

-	Reactor多线程模型

![Reactor多线程模型](https://static.oschina.net/uploads/img/201510/20083315_ObVg.png "Reactor多线程模型")

使用一个主线程，该线程的Selector只负责接收连接的建立，建立连接后将该连接的读写事件注册到其他线程的Selector，目前大部分都是采用这种方式。

下面再举几个例子：ZooKeeper通信使用的线程模型、Netty、Gecko

###2.2.1 ZooKeeper通信使用的线程模型

ZooKeeper通信使用的线程模型就比较简单，默认使用上述Reacter单线程模型。并且是采用jdk自带的NIO类库来实现的。

服务器端开启一个线程，创建出一个Selector，在该线程中不断的检测连接的建立和读写事件。代码大致如下

![ZooKeeper NIO通讯](https://static.oschina.net/uploads/img/201510/29084554_s23M.png "ZooKeeper NIO通讯")

![ZooKeeper NIO通讯](https://static.oschina.net/uploads/img/201510/29084702_84Lh.png "ZooKeeper NIO通讯")

一旦有客户端建立连接，则获取SocketChannel，并设置非阻塞，注册到seclector上，绑定到SelectionKey

一旦是读写事件则从SelectionKey中取出进行数据的读写

上述事件都是由一个线程来完成的，要求不高的情况下就可以满足了。如大部分框架使用ZooKeeper的场景都是作为一个协调服务，访问量很小的。如果ZooKeeper承载了很多很多的服务，上述单线程通信方式满足不了了，就需要使用Reactor多线程模型，如采用netty作为NIO通讯框架。

###2.2.2 Netty采用的线程模型

可以简单就说成采用Reactor多线程模型。会有一个boss线程池和worker线程池。boss线程池上的Selector专门负责连接的建立，一旦连接建立起来之后，从worker线程池上获取一个线程（获取过程实现简单的负载均衡），将连接注册到该线程上的Selector上，让worker线程池负责读写操作，具体如下：

-	1 每一个NioEventLoop：包含一个线程和一个Selector selector复用器

	因此一个NioEventLoop就可以处理很多链路，每个链路的读写操作全部交给这一个线程来处理，避免了并发操作同一个链路的可能性

	![NioEventLoop处理](https://static.oschina.net/uploads/img/201511/01202038_1gFO.png "NioEventLoop处理")

-	2 每当有一个新的客户端接入，则从NioEventLoop线程组中顺序获取一个可用的NioEventLoop，当到达数组上限之后，重新返回到0，通	过这种方式，可以基本保证各个NioEventLoop的负载均衡。一个客户端连接只注册到一个NioEventLoop上，这样就避免了多个IO线程去	并发操作它

###2.2.3 Gecko采用的线程模型

也是采用的是Reactor多线程模型。有一个SelectorManager，它含一个Reactor[] reactorSet，每一个Reactor都是一个线程，并且拥有自己的Selector，和上述netty的NioEventLoop差不多。

SelectorManager将第一个Reactor专门用作接收客户端连接的作用，连接建立起来之后，将连接注册到其余的Reactor的Selector上。 
在选择Reactor也是采用简单的顺序轮训的策略。

SelectorManager代码如下：

![SelectorManager](https://static.oschina.net/uploads/img/201511/22204833_ANbX.png "SelectorManager")

选择Reactor代码如下：

![输入图片说明](https://static.oschina.net/uploads/img/201511/22205031_nt0F.png "在这里输入图片标题")

上述netty中，有关某个连接的读写事件全部交给了该连接注册的worker线程上，对某个连接的操作只会在同一个线程中进行，避免了并发操作，但是每个线程必须完成当前的读写操作后才能去执行接下来的另外的读写事件。

而Gecko则是将读写事件交给了专门的读写线程池，这样的话Reactor线程只管发出读写任务（一个Runnable对象），真正的读写操作交给读写线程池来完成，Reactor线程处理事件的并发量就大了，但是这样的话就是就可能出现多个IO线程并发操作同一个连接，加大了出错的风险。

##2.3 编解码和序列化反序列化的处理

这一部分就需要处理粘包问题、采用的协议。不同的协议就要使用不同编解码器，所以文章开头所说的可插拔的协议设计就是指用户可以自己指定自己协议所使用的编解码器

就来详细看下处理粘包的过程：

-	1 从SocketChannel中取出全部的可读数据，将数据写到一个buffer中，如果buffer承载不下，暂时就不读了，先处理buffer中的数据，处理完成之后再来写剩余的数据到buffer中

-	2 将buffer执行flip操作，由写状态进入读状态

-	3 buffer中有很多数据，可能是客户端发送过来的多条消息数据，而目前采用的大部分协议都是固定长度的Header加Body的形式，Header中指明了Body的长度。这样的话，肯能就存在几种情况：

	-	3.1 buffer中可读长度不足Header的固定长度，则此数据还没到齐，暂时还不能解析，需要直接返回，不做处理
	-	3.2 buffer中可读长度大于一个Header的固定长度，则可以进行读取解析，获取Header中每一位定义的信息（这就是所采用的协议定义的），如dubbo协议中Header中的每一位信息

	![dubbo协议](https://static.oschina.net/uploads/img/201510/31090546_37Uy.png "dubbo协议")

	-	3.3 Header中读取完毕，就知道了Body的长度，还需要判断下buffer中剩余可读数据的长度是否大于刚才解析出的Header中给出的Body的长度，如果不足的话，此时有两种方式：
	
		一种就是回溯所读取的数据，在解析Header之前，先保留当前的index，解析header之后，发现buffer中剩余长度不足则将buffer重置读取位置到上述index。

		另一种就是将解析出来的Header信息先绑定到连接上，每次解析数据前，先从连接中获取Header信息，如果有，说明上次已经解析完Header信息了，但是Body中信息不足，那就直接开始解析Body了

		第一种方式：一旦出现Body信息不足，就回溯了，当信息充足后，会重复解析Header中的内容。第二种方式则不会进行重复解析

		而dubbo中处理方式就是采用的第一种，保留一个index。一旦消息不完整则回退到该index。

		目前Gecko就是采用的第二种方式，将Header先暂时保存在连接中

		
来详细看下dubbo中的处理方式：

![dubbo中对粘包消息的处理](https://static.oschina.net/uploads/img/201511/22224214_Ky1d.png "dubbo中对粘包消息的处理")


再详细看下Gecko对粘包的处理方式：

![Gecko对粘包的处理方式](https://static.oschina.net/uploads/img/201511/22230052_4CBc.png "Gecko对粘包的处理方式")


buffer中多条消息的处理：

![buffer中多条消息的处理](https://static.oschina.net/uploads/img/201511/22230540_wVGF.png "buffer中多条消息的处理")

来看下一个解码器的实现，启动不同场景下的Server使用不同的编解码器，如metaq命令行方式就是使用如下编解码器，专门用于处理请求命令的：

![Gecko中对Header和Body的解析](https://static.oschina.net/uploads/img/201511/22222912_141f.png "Gecko中对Header和Body的解析")

对于RPC，不仅要进行编解码，同时还要涉及序列化和发序列化的问题：

dubbo中都是通过url中参数值来定制化编解码器和序列化反序列化方式,提供了各种丰富多样的编解码和序列化反序列化方式，而Gecko实现的RPC也可以实现定制化。默认方式是使用jdk自带的序列化和反序列化方式，即ObjectInputStream方式，简单代码如下

Gecko RPC解码过程：

![RPC解码过程](https://static.oschina.net/uploads/img/201511/23081934_HyMV.png "RPC解码过程")

Gecko RPC序列化过程:

![Gecko RPC序列化过程](https://static.oschina.net/uploads/img/201511/23082218_xfJk.png "Gecko RPC序列化过程")

##2.4 重连管理


##2.5 调用方式：同步转异步、异步回调


##2.6 分组管理和负载均衡





