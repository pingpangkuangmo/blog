#1 系列目录

-	[dubbo源码分析系列（1）扩展机制的实现](http://my.oschina.net/pingpangkuangmo/blog/508963)
-	[dubbo源码分析系列（2）服务的发布](http://my.oschina.net/pingpangkuangmo/blog/511766)
-	[dubbo源码分析系列（3）服务的引用](http://my.oschina.net/pingpangkuangmo/blog/515673)

#2 NIO通信层的抽象

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

用一个boss线程，创建Selector，用于不断监听Socket连接、客户端的读写操作等

用一个线程池即workers，负责处理Selector派发的读写操作。

由于boss线程可以接收更多的Socket连接，同时可以充分利用线程池中的每个线程，减少了BIO模型下每个线程为单独的socket的等待时间。

##2.2 服务器端如何集成netty和mina

先来简单总结下上述netty和mina的相似之处，然后进行抽象概括成接口

-	1 各自有各自的编程启动方式
-	2 都需要各自的ChannelHandler实现，用于处理各自的Channel或者IoSession的连接、读写等事件。

	对于netty来说： 需要继承org.jboss.netty.channel.SimpleChannelHandler（或者其他方式），来处理org.jboss.netty.channel.Channel的连接读写事件

	对于mina来说：需要继承org.apache.mina.common.IoHandlerAdapter（或者其他方式），来处理org.apache.mina.common.IoSession的连接读写事件

为了统一上述问题，dubbo需要做如下事情：

-	1 定义dubbo的com.alibaba.dubbo.remoting.Channel接口

	-	1.1 针对netty，上述Channel的实现为NettyChannel，内部含有一个netty自己的org.jboss.netty.channel.Channel channel对象，即该com.alibaba.dubbo.remoting.Channel接口的功能实现全部委托为底层的org.jboss.netty.channel.Channel channel对象来实现

	-	1.2 针对mina，上述Channel实现为MinaChannel，内部包含一个mina自己的org.apache.mina.common.IoSession session对象，即该com.alibaba.dubbo.remoting.Channel接口的功能实现全部委托为底层的org.apache.mina.common.IoSession session对象来实现


-	2 定义自己的com.alibaba.dubbo.remoting.ChannelHandler接口，用于处理com.alibaba.dubbo.remoting.Channel接口的连接读写事件，如下所示

		public interface ChannelHandler {
	
		    void connected(Channel channel) throws RemotingException;
		
		    void disconnected(Channel channel) throws RemotingException;
		
		    void sent(Channel channel, Object message) throws RemotingException;
		
		    void received(Channel channel, Object message) throws RemotingException;
		
		    void caught(Channel channel, Throwable exception) throws RemotingException;
	
		}

	-	2.1 先定义用于处理netty的NettyHandler，需要按照netty的方式继承netty的org.jboss.netty.channel.SimpleChannelHandler，此时NettyHandler就可以委托dubbo的com.alibaba.dubbo.remoting.ChannelHandler接口实现来完成具体的功能，在交给com.alibaba.dubbo.remoting.ChannelHandler接口实现之前，需要先将netty自己的org.jboss.netty.channel.Channel channel转化成上述的NettyChannel，见NettyHandler

			public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
		        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
		        try {
		            if (channel != null) {
		                channels.put(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()), channel);
		            }
		            handler.connected(channel);
		        } finally {
		            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
		        }
		    }
		
		    @Override
		    public void channelDisconnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
		        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
		        try {
		            channels.remove(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()));
		            handler.disconnected(channel);
		        } finally {
		            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
		        }
		    }

	-	2.2 先定义用于处理mina的MinaHandler，需要按照mina的方式继承mina的org.apache.mina.common.IoHandlerAdapter，此时MinaHandler就可以委托dubbo的com.alibaba.dubbo.remoting.ChannelHandler接口实现来完成具体的功能，在交给com.alibaba.dubbo.remoting.ChannelHandler接口实现之前，需要先将mina自己的org.apache.mina.common.IoSession转化成上述的MinaChannel，见MinaHandler

			public void sessionOpened(IoSession session) throws Exception {
		        MinaChannel channel = MinaChannel.getOrAddChannel(session, url, handler);
		        try {
		            handler.connected(channel);
		        } finally {
		            MinaChannel.removeChannelIfDisconnectd(session);
		        }
		    }
		
		    @Override
		    public void sessionClosed(IoSession session) throws Exception {
		        MinaChannel channel = MinaChannel.getOrAddChannel(session, url, handler);
		        try {
		            handler.disconnected(channel);
		        } finally {
		            MinaChannel.removeChannelIfDisconnectd(session);
		        }
		    }

做了上述事情之后，全部逻辑就统一到dubbo自己的com.alibaba.dubbo.remoting.ChannelHandler接口如何来处理自己的com.alibaba.dubbo.remoting.Channel接口。

这就需要看下com.alibaba.dubbo.remoting.ChannelHandler接口的实现有哪些：

![ChannelHandler接口实现](https://static.oschina.net/uploads/img/201510/23082021_ZCfI.png "ChannelHandler接口实现")


-	3 定义Server接口用于统一大家的启动流程

	先来看下整体的Server接口实现情况

	![Server接口实现情况](https://static.oschina.net/uploads/img/201510/24081243_Vrnj.png "Server接口实现情况")


	如NettyServer的启动流程： 按照netty自己的API启动方式，然后依据外界传递进来的com.alibaba.dubbo.remoting.ChannelHandler接口实现，创建出NettyHandler，最终对用户的连接请求的处理全部交给NettyHandler来处理，NettyHandler又交给了外界传递进来的com.alibaba.dubbo.remoting.ChannelHandler接口实现。

	至此就将所有底层不同的通信实现全部转化到了外界传递进来的com.alibaba.dubbo.remoting.ChannelHandler接口的实现上了。

	而上述Server接口的另一个分支实现HeaderExchangeServer则充当一个装饰器的角色，为所有的Server实现增添了如下功能：

	向该Server所有的Channel依次进行心跳检测：

	-	如果当前时间减去最后的读取时间大于heartbeat时间或者当前时间减去最后的写时间大于heartbeat时间，则向该Channel发送一次心跳检测
	-	如果当前时间减去最后的读取时间大于heartbeatTimeout，则服务器端要关闭该Channel，如果是客户端的话则进行重新连接（客户端也会使用这个心跳检测任务）

##2.3 客户端如何集成netty和mina

服务器端了解了之后，客户端就也非常清楚了，整体类图如下：

![Client接口实现情况](https://static.oschina.net/uploads/img/201510/24084841_7w2i.png "Client接口实现情况")


如NettyClient在使用netty的API开启客户端之后，仍然使用NettyHandler来处理。还是最终转化成com.alibaba.dubbo.remoting.ChannelHandler接口实现上了。

HeaderExchangeClient和上面的HeaderExchangeServer非常类似，就不再提了。


我们可以看到这样集成完成之后，就完全屏蔽了底层通信细节，将逻辑全部交给了com.alibaba.dubbo.remoting.ChannelHandler接口的实现上了。从上面我们也可以看到，该接口实现也会经过层层装饰类的包装，才会最终交给底层通信。

如HeartbeatHandler装饰类：

	public void sent(Channel channel, Object message) throws RemotingException {
        setWriteTimestamp(channel);
        handler.sent(channel, message);
    }

    public void received(Channel channel, Object message) throws RemotingException {
        setReadTimestamp(channel);
        if (isHeartbeatRequest(message)) {
            Request req = (Request) message;
            if (req.isTwoWay()) {
                Response res = new Response(req.getId(), req.getVersion());
                res.setEvent(Response.HEARTBEAT_EVENT);
                channel.send(res);
                if (logger.isInfoEnabled()) {
                    int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                    if(logger.isDebugEnabled()) {
                        logger.debug("Received heartbeat from remote channel " + channel.getRemoteAddress()
                                        + ", cause: The channel has no data-transmission exceeds a heartbeat period"
                                        + (heartbeat > 0 ? ": " + heartbeat + "ms" : ""));
                    }
	            }
            }
            return;
        }
        if (isHeartbeatResponse(message)) {
            if (logger.isDebugEnabled()) {
            	logger.debug(
                    new StringBuilder(32)
                        .append("Receive heartbeat response in thread ")
                        .append(Thread.currentThread().getName())
                        .toString());
            }
            return;
        }
        handler.received(channel, message);
    }

就会拦截那些上述提到的心跳检测请求。更新该Channel的最后读写时间。


##2.4 同步调用和异步调用的实现

首先设想一下我们目前的通信方式，使用netty mina等异步事件驱动的通信框架，将Channel中信息都分发到Handler中去处理了，Handler中的send方法只负责不断的发送消息，receive方法只负责不断接收消息，这时候就产生一个问题：

客户端如何对应同一个Channel的接收的消息和发送的消息之间的匹配呢？

这也很简单，就需要在发送消息的时候，必须要产生一个请求id，将调用的信息连同id一起发给服务器端，服务器端处理完毕后，再将响应信息和上述请求id一起发给客户端，这样的话客户端在接收到响应之后就可以根据id来判断是针对哪次请求的响应结果了。

来看下DubboInvoker中的具体实现

	boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
    boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
    int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
    if (isOneway) {
    	boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
        currentClient.send(inv, isSent);
        RpcContext.getContext().setFuture(null);
        return new RpcResult();
    } else if (isAsync) {
    	ResponseFuture future = currentClient.request(inv, timeout) ;
        RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
        return new RpcResult();
    } else {
    	RpcContext.getContext().setFuture(null);
        return (Result) currentClient.request(inv, timeout).get();
    }


-	如果不需要返回值，直接使用send方法，发送出去，设置当期和线程绑定RpcContext的future为null
-	如果需要异步通信，使用request方法构建一个ResponseFuture，然后设置到和线程绑定RpcContext中
-	如果需要同步通信，使用request方法构建一个ResponseFuture，阻塞等待请求完成

可以看到的是它把ResponseFuture设置到与当前线程绑定的RpcContext中了，如果我们要获取异步结果，则需要通过RpcContext来获取当前线程绑定的RpcContext，然后就可以获取Future对象。如下所示：

	String result1 = helloService.hello("World");
    System.out.println("result :"+result1);
    System.out.println("result : "+RpcContext.getContext().getFuture().get());

当设置成异步请求的时候，result1则为null,然后通过RpcContext来获取相应的值。

然后我们来看下异步请求的整个实现过程，即上述currentClient.request方法的具体内容：

	public ResponseFuture request(Object request, int timeout) throws RemotingException {
        // create request.
        Request req = new Request();
        req.setVersion("2.0.0");
        req.setTwoWay(true);
        req.setData(request);
        DefaultFuture future = new DefaultFuture(channel, req, timeout);
        try{
            channel.send(req);
        }catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }

-	第一步：创建出一个request对象，创建过程中就自动产生了requestId,如下

		public class Request {
    		private final long    mId;
    		private static final AtomicLong INVOKE_ID = new AtomicLong(0);

			public Request() {
		        mId = newId();
		    }

			private static long newId() {
		        // getAndIncrement()增长到MAX_VALUE时，再增长会变为MIN_VALUE，负数也可以做为ID
		        return INVOKE_ID.getAndIncrement();
		    }
		}

-	第二步：根据request请求封装成一个DefaultFuture对象，通过该对象的get方法就可以获取到请求结果。该方法会阻塞一直到请求结果产生。同时DefaultFuture对象会被存至DefaultFuture类如下结构中：

		private static final Map<Long, DefaultFuture> FUTURES   = new ConcurrentHashMap<Long, DefaultFuture>();

	key就是请求id

-	第三步：将上述请求对象发送给服务器端，同时将DefaultFuture对象返给上一层函数，即DubboInvoker中，然后设置到当前线程中

-	第四步：用户通过RpcContext来获取上述DefaultFuture对象来获取请求结果，会阻塞至服务器端返产生结果给客户端

-	第五步：服务器端产生结果，返回给客户端会在客户端的handler的receive方法中接收到，接收到之后判别接收的信息是Response后，

		static void handleResponse(Channel channel, Response response) throws RemotingException {
	        if (response != null && !response.isHeartbeat()) {
	            DefaultFuture.received(channel, response);
	        }
	    }

	就会根据response的id从上述FUTURES结构中查出对应的DefaultFuture对象，并把结果设置进去。此时DefaultFuture的get方法则不再阻塞，返回刚刚设置好的结果。


至此异步通信大致就了解了，但是我们会发现一个问题：

当某个线程多次发送异步请求时，都会将返回的DefaultFuture对象设置到当前线程绑定的RpcContext中，就会造成了覆盖问题，如下调用方式：

	String result1 = helloService.hello("World");
    String result2 = helloService.hello("java");
    System.out.println("result :"+result1);
    System.out.println("result :"+result2);
    System.out.println("result : "+RpcContext.getContext().getFuture().get());
    System.out.println("result : "+RpcContext.getContext().getFuture().get());

即异步调用了hello方法，再次异步调用，则前一次的结果就被冲掉了，则就无法获取前一次的结果了。必须要调用一次就立马将DefaultFuture对象获取走，以免被冲掉。即这样写：

	String result1 = helloService.hello("World");
    Future<String> result1Future=RpcContext.getContext().getFuture();
    String result2 = helloService.hello("java");
    Future<String> result2Future=RpcContext.getContext().getFuture();
    System.out.println("result :"+result1);
    System.out.println("result :"+result2);
    System.out.println("result : "+result1Future.get());
    System.out.println("result : "+result2Future.get());


#3 通信层与dubbo的结合

从上面可以了解到如何对不同的通信框架进行抽象，屏蔽底层细节，统一将逻辑交给ChannelHandler接口实现来处理。然后我们就来了解下如何与dubbo的业务进行对接，也就是在什么时机来使用上述通信功能：

##3.1 服务的发布过程使用通信功能

如DubboProtocol在发布服务的过程中：

-	1 DubboProtocol中有一个如下结构

		Map<String, ExchangeServer> serverMap

	在发布一个服务的时候会先根据服务的url获取要发布的服务所在的host和port，以此作为key来从上述结构中寻找是否已经有对应的ExchangeServer（上面已经说明）。

-	2 如果没有的话，则会创建一个，创建过程如下：

		ExchangeServer server = Exchangers.bind(url, requestHandler);

	其中requestHandler就是DubboProtocol自身实现的ChannelHandler。
	
	获取一个ExchangeServer，它的实现主要是Server的装饰类，依托外部传递的Server来实现Server功能，而自己加入一些额外的功能，如ExchangeServer的实现HeaderExchangeServer，就是加入了心跳检测的功能。

	所以此时我们可以自定义扩展功能来实现Exchanger。接口定义如下：

		@SPI(HeaderExchanger.NAME)
		public interface Exchanger {
		
		    @Adaptive({Constants.EXCHANGER_KEY})
		    ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException;
		
		    @Adaptive({Constants.EXCHANGER_KEY})
		    ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException;
		
		}

	默认使用的就是HeaderExchanger，它创建的ExchangeServer是HeaderExchangeServer如下所示：

		public class HeaderExchanger implements Exchanger {
    
		    public static final String NAME = "header";
		
		    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
		        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
		    }
		
		    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
		        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
		    }
		
		}

	HeaderExchangeServer仅仅是一个Server接口的装饰类，需要依托外部传递Server实现来完成具体的功能。此Server实现可以是netty也可以是mina等。所以我们可以自定义Transporter实现来选择不同底层通信框架，接口定义如下：

		@SPI("netty")
		public interface Transporter {
		
		    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
		    Server bind(URL url, ChannelHandler handler) throws RemotingException;
		
		    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
		    Client connect(URL url, ChannelHandler handler) throws RemotingException;
		
		}
	默认采用netty实现，如下：

		public class NettyTransporter implements Transporter {

		    public static final String NAME = "netty";
		    
		    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
		        return new NettyServer(url, listener);
		    }
		
		    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
		        return new NettyClient(url, listener);
		    }
		
		}
	至此就到了我们上文介绍的内容了。同时DubboProtocol的ChannelHandler实现经过层层装饰器包装，最终传给底层通信Server。

	客户端发送请求给服务器端时，底层通信Server会将请求经过层层处理最终传递给DubboProtocol的ChannelHandler实现，在该实现中，会根据请求参数找到对应的服务器端本地Invoker，然后执行，再将返回结果通过底层通信Server发送给客户端。
	

##3.2 客户端的引用服务使用通信功能

在DubboProtocol引用服务的过程中：

-	1 使用如下方式创建client

		ExchangeClient client=Exchangers.connect(url ,requestHandler)；

	requestHandler还是DubboProtocol中ChannelHandler实现。

	和Server类似，我们可以通过自定义Exchanger实现来创建出不同功能的ExchangeClient。默认的Exchanger实现是HeaderExchanger

		public class HeaderExchanger implements Exchanger {
    
		    public static final String NAME = "header";
		
		    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
		        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
		    }
		
		    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
		        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
		    }
		
		}

	创建出来的ExchangeClient是HeaderExchangeClient，它也是Client的包装类，仅仅在Client外层加上心跳检测的功能，向它所连接的服务器端发送心跳检测。

	HeaderExchangeClient需要外界给它传一个Client实现，这是由Transporter接口实现来定的，默认是NettyTransporter

		public class NettyTransporter implements Transporter {

		    public static final String NAME = "netty";
		    
		    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
		        return new NettyServer(url, listener);
		    }
		
		    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
		        return new NettyClient(url, listener);
		    }
		
		}

	创建出来的的Client实现是NettyClient。

	同时DubboProtocol的ChannelHandler实现经过层层装饰器包装，最终传给底层通信Client。

	客户端的DubboInvoker调用远程服务的时候，会将调用信息通过ExchangeClient发送给服务器端，然后返回一个ResponseFuture，根据客户端选择的同步还是异步方式，来决定阻塞还是直接返回，这一部分在上文同步调用和异步调用的实现中已经详细说过了。

#4 结束语

本篇文章主要介绍了集成netty和mina的那一块的通信接口及实现的设计，下篇主要介绍编解码的过程