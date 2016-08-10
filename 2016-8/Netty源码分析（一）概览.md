准备将Netty的源码过一下，一来对自己是个总结消化的过程，二来希望对那些打算看Netty源码的人（已经熟悉Netty的Reactor模型）能有一些帮助。目前所看Netty版本是4.1.3.Final。

# 1 目录

[Netty概览]()

# 2 概览

## 2.1 服务器端demo

看下一个简单的Netty服务器端的例子

	public static void main(String[] args){
		EventLoopGroup bossGroup=new NioEventLoopGroup(1);
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap serverBootstrap=new ServerBootstrap();
			serverBootstrap.group(bossGroup,workerGroup)
				.channel(NioServerSocketChannel.class)
				.option(ChannelOption.SO_BACKLOG, 200)
				.childHandler(new ChannelInitializer<SocketChannel>() {
					@Override
					protected void initChannel(SocketChannel ch) throws Exception {
						ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(80,0,4,0,4));
						ch.pipeline().addLast(new StringDecoder(Charset.forName("UTF-8")));
						ch.pipeline().addLast(new TcpServerHandler());
					}
				});
			ChannelFuture f=serverBootstrap.bind(8080).sync();
			f.channel().closeFuture().sync();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}finally {  
            workerGroup.shutdownGracefully();  
            bossGroup.shutdownGracefully();  
        }  
	}

先来简单说说上述遇到的类：

## 2.2 EventLoopGroup介绍

它主要包含2个方面的功能，注册Channel和执行一些Runnable任务。

![EventLoopGroup介绍](https://static.oschina.net/uploads/img/201608/06184758_9ZPc.png "EventLoopGroup介绍")

**功能1：先来看看注册Channel**，即将Channel注册到Selector上，由Selector来调度Channel的相关事件，如读、写、Accept等事件。

而EventLoopGroup的设计是，它包含多个EventLoop（每一个EventLoop通常内部包含一个线程），在执行上述注册过程中是需要选择其中的一个EventLoop来执行上述注册行为，这里就出现了一个选择策略的问题，该选择策略接口是EventExecutorChooser，你也可以自定义一个实现。

从上面可以看到，EventLoopGroup做的工作大部分是一些总体性的工作如初始化上述多个EventLoop、EventExecutorChooser等，具体的注册Channel还是交给它内部的EventLoop来实现。

**功能2：执行一些Runnable任务**

EventLoopGroup继承了EventExecutorGroup，EventExecutorGroup也是EventExecutor的集合，EventExecutorGroup也是掌管着EventExecutor的初始化工作，EventExecutorGroup对于Runnable任务的执行也是选择内部中的一个EventExecutor来做具体的执行工作。

netty中很多任务都是异步执行的，一旦当前线程要对某个EventLoop执行相关操作，如注册Channel到某个EventLoop，如果当前线程和所要操作的EventLoop内部的线程不是同一个，则当前线程就仅仅向EventLoop提交一个注册任务，对外返回一个ChannelFuture。

## 2.3 ChannelPipeline介绍

上述EventLoopGroup可以将一个Channel注册到内部的一个EventLoop的Selector上，然后对于这个Channel的相关读写等事件，Netty专门设计了一个ChannelPipeline来进行处理。每一个Channel都有一个ChannelPipeline来处理该Channel的读写等事件。

![ChannelPipeline介绍](https://static.oschina.net/uploads/img/201608/06193041_FLD8.png "ChannelPipeline介绍")

## 2.4 bind过程

上述serverBootstrap的bind过程如下：

-	创建出你所指定的NioServerSocketChannel，然后初始化一些Socket方面的参数

-	为上述Channel的ChannelPipeline配置一个ChannelHandler，该ChannelHandler的作用就是在该Channel成功注册到Selector上的时候，初始化一些逻辑，即initChannel方法中执行一些逻辑，该逻辑就是向ChannelPipeline中添加一个新的ChannelHandler即ServerBootstrapAcceptor

-	然后开始将该Channel注册到上述EventLoopGroup bossGroup中，该EventLoopGroup bossGroup会选择内部的一个EventLoop来执行实际的注册行为（这个时候就是当前线程和操作的EventLoop不是同一个线程，即该过程是异步提交一个Runnable），一旦注册完成，就执行上述ChannelHandler的initChannel方法


至此，就完成了整个bind过程。一旦EventLoop内部的Selector检测到NioServerSocketChannel有新的连接到来的事件，则会交给NioServerSocketChannel的ChannelPipeline来处理，重点就是ChannelPipeline中的上述ServerBootstrapAcceptor，ServerBootstrapAcceptor做如下操作：

-	1 为新的Channel的ChannelPipeline配置我们上述代码中的childHandler指定的ChannelHandler

-	2 将新的Channel注册到了上述EventLoopGroup workerGroup中

## 2.5 sync介绍

bind方法返回的是一个ChannelFuture，从上面我们也知道该过程是异步的，sync方法则是一直等待到该异步过程结束。

再看下f.channel().closeFuture().sync()这个方法

每一个ChannelFuture都是和一个Channel绑定的，所以可以通过ChannelFuture来获取对应绑定的Channel对象

每一个Channel对象都有一个CloseFuture closeFuture对象，上述closeFuture方法并不是去执行close方法而是获取到这个CloseFuture closeFuture对象，然后调用它的sync方法即等待这个Future的结束。一般正常情况下是不会调用这个Future的结束方法的，只是在上述过程或者其他过程出现问题的时候，如注册到EventLoop失败等才会去调用这个Feture的结束方法，所以正常情况下主线程会一直阻塞在CloseFuture closeFuture的sync方法上。

# 3 总结

