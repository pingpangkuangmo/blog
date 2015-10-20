#1 系列目录

-	[dubbo源码分析系列（1）扩展机制的实现](http://my.oschina.net/pingpangkuangmo/blog/508963)
-	[dubbo源码分析系列（2）服务的发布](http://my.oschina.net/pingpangkuangmo/blog/511766)
-	[dubbo源码分析系列（3）服务的引用](http://my.oschina.net/pingpangkuangmo/blog/515673)

#2 NIO通信

目前dubbo已经集成的有netty、mina、grizzly。先来通过案例简单了解下netty、mina编程（grizzly没有了解过）

##2.1 netty和mina的简单案例

netty原本是jboss开发的，后来单独出来了，所以会有两种版本就是org.jboss.netty和io.netty两种包类型的，而dubbo内置的是前者。目前还不是很熟悉，可能稍有差别，但是整体大概都是一样的。

我们先来看下io.netty的案例：

	public static void main(String[] args){
		EventLoopGroup bossGroup=new NioEventLoopGroup();
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap serverBootstrap=new ServerBootstrap();
			serverBootstrap.group(bossGroup,workerGroup)
				.channel(NioServerSocketChannel.class)
				.childHandler(new ChannelInitializer<SocketChannel>() {
					@Override
					protected void initChannel(SocketChannel ch) throws Exception {
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

mina的案例：

	public static void main(String[] args) throws IOException{
		IoAcceptor acceptor = new NioSocketAcceptor();
		acceptor.getFilterChain().addLast("codec",new ProtocolCodecFilter(
				new TextLineCodecFactory(Charset.forName("UTF-8"),"\r\n", "\r\n")));
		acceptor.setHandler(new TcpServerHandler());  
        acceptor.bind(new InetSocketAddress(8080));  
	}

两者都是使用Reactor模型结构。而最原始BIO模型如下：

![原始BIO模型](https://static.oschina.net/uploads/img/201510/20083738_I5mX.png "原始BIO模型")

每来一个Socket连接都为该Socket创建一个线程来处理。由于总线程数有限制，导致Socket连接受阻，所以BIO模型并发量并不大

Rector多线程模型如下，更多信息见[Netty系列之Netty线程模型](http://www.infoq.com/cn/articles/netty-threading-model)：

![Rector多线程模型](https://static.oschina.net/uploads/img/201510/20083315_ObVg.png "Rector多线程模型")

用一个boss线程，创建Selector，用于不断监听Socket连接、客户端的读写操作等。

用一个线程池即workers，负责处理Selector派发的读写操作。

由于boss线程可以无限的接收Socket连接，


