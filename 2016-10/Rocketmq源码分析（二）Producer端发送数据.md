# 1 系列

-	[整体架构图](https://my.oschina.net/pingpangkuangmo/blog/753742)
-	[producer端发送消息](https://my.oschina.net/pingpangkuangmo/blog/755579)
-	broker端接收消息
-	broker端消息的存储
-	consumer消费消息
-	分布式事务的实现
-	定时消息的实现
-	关于顺序消费话题
-	关于重复消息话题
-	关于高可用话题

# 2 发送消息案例

下面分别给出发送消息的3个官方案例

## 2.1 同步发送

![同步发送](https://static.oschina.net/uploads/img/201610/08112739_Osbj.png "同步发送")

同步发送直接得到响应结果

## 2.2 异步发送

![异步发送](https://static.oschina.net/uploads/img/201610/08113010_CSKN.png "异步发送")

在发送的时候需要指定一个SendCallback回调，用于处理发送成功和发送异常的结果

## 2.3 顺序发送

![顺序发送](https://static.oschina.net/uploads/img/201610/08113528_ibmk.png "顺序发送")

在发送的时候需要指定一个MessageQueueSelector，用于选择将该消息发送到哪一个消息队列。可以看到顺序发送内部也是采用同步来发送的

下面就来探讨下Producer要解决的问题

# 3 Producer的源码介绍

## 3.1 通信层的设计

RocketMQ采用RemotingClient来实现底层的通信

### 3.1.1 三种发送方式

RemotingClient定义了如下三种通信方式：

同步发送：

	public RemotingCommand invokeSync(final String addr, final RemotingCommand request,
                                      final long timeoutMillis)

异步发送：

	public void invokeAsync(final String addr, final RemotingCommand request, final long timeoutMillis,
                            final InvokeCallback invokeCallback)

单方向发送（不需要知道响应）：

	public void invokeOneway(final String addr, final RemotingCommand request, final long timeoutMillis)

这里发送请求的数据是RemotingCommand，会将RemotingCommand编码成字节数组，服务器端对字节数组进行解码成RemotingCommand，然后处理，得到响应结果为RemotingCommand。

上述RemotingClient接口的实现是NettyRemotingClient，采用Netty的客户端Bootstrap来实现。

需要做的就是选择EventLoopGroup、实现编解码器，以及实现Netty的ChannelHandler

代码片段如下：

	this.eventLoopGroupWorker = new NioEventLoopGroup(1, new ThreadFactory() {
            private AtomicInteger threadIndex = new AtomicInteger(0);


            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, String.format("NettyClientSelector_%d", this.threadIndex.incrementAndGet()));
            }
        });

	Bootstrap handler = this.bootstrap.group(this.eventLoopGroupWorker).channel(NioSocketChannel.class)//
                //
                .option(ChannelOption.TCP_NODELAY, true)
                //
                .option(ChannelOption.SO_KEEPALIVE, false)
                //
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, nettyClientConfig.getConnectTimeoutMillis())
                //
                .option(ChannelOption.SO_SNDBUF, nettyClientConfig.getClientSocketSndBufSize())
                //
                .option(ChannelOption.SO_RCVBUF, nettyClientConfig.getClientSocketRcvBufSize())
                //
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(//
                                defaultEventExecutorGroup, //
                                new NettyEncoder(), //
                                new NettyDecoder(), //
                                new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()), //
                                new NettyConnetManageHandler(), //
                                new NettyClientHandler());
                    }
                });


这里选择的EventLoopGroup是NioEventLoopGroup，编解码器是NettyEncoder、NettyDecoder。这2个编解码器就是用于上述的RemotingCommand和字节数组之间的转换的。

再来看看上述Netty的ChannelHandler的实现NettyClientHandler

### 3.1.2 NettyClientHandler

它将读到的数据分成2类，一类是服务器端返回的响应数据，另一类是服务器端请求客户端的数据，如下所示

	public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        final RemotingCommand cmd = msg;
        if (cmd != null) {
            switch (cmd.getType()) {
                case REQUEST_COMMAND:
                    processRequestCommand(ctx, cmd);
                    break;
                case RESPONSE_COMMAND:
                    processResponseCommand(ctx, cmd);
                    break;
                default:
                    break;
            }
        }
    }

根据RemotingCommand的类型标识位，如果是REQUEST_COMMAND类型，表明是服务器端请求客户端的，如果是RESPONSE_COMMAND表明是服务器端返回的响应数据。

如果是REQUEST_COMMAND：客户端需要根据服务器端的请求码，找到对应的处理函数以及线程池来进行处理，处理完成之后返回响应给服务器端

NettyClientHandler中有这样的一个数据结构

	HashMap<Integer/* request code */, Pair<NettyRequestProcessor, ExecutorService>> processorTable

存储着请求码对应的处理器NettyRequestProcessor，以及该处理器对应的线程池ExecutorService。也就是说服务器端请求客户端执行某个操作，客户端会拿出对应的处理器在对应的线程池中来执行相应的处理。


如果是RESPONSE_COMMAND：客户端根据参数RemotingCommand得到请求id，根据这个请求id找到之前的请求，这里是ResponseFuture，设置这个ResponseFuture的状态为执行结束。并通过判断这个ResponseFuture是否有回调函数，如果有则执行回调函数。

对于上述的同步发送：创建了一个ResponseFuture，并将请求id和ResponseFuture的映射关系存放在

	ConcurrentHashMap<Integer /* opaque */, ResponseFuture> responseTable

NettyClientHandler的上述responseTable中，然后客户端等待响应，等待超时时间是可以指定的。一旦服务器端有响应则会走上述RESPONSE_COMMAND处理流程，此时客户端得到响应结果不再阻塞

对于上述异步发送：创建了一个ResponseFuture，这里会将异步发送的InvokeCallback放到ResponseFuture中，并将请求id和ResponseFuture的映射关系存放在responseTable中，客户端不阻塞，直接返回。一旦服务器端有响应则会走上述RESPONSE_COMMAND处理流程，会执行前面设置的InvokeCallback

对于上述单方向发送：不创建ResponseFuture。


大部分的同步、异步、单方向的通信都是采用上述类似的方式来实现的。

## 3.2 API的实现

上述RemotingClient负责通信实现，而MQClientAPIImpl则是利用RemotingClient来实现RocketMQ的相关功能，把具体的功能需求转化成RemotingCommand，交给RemotingClient去执行，如创建topic

	public void createTopic(final String addr, final String defaultTopic, final TopicConfig topicConfig, final long timeoutMillis)
            throws RemotingException, MQBrokerException, InterruptedException, MQClientException {
        CreateTopicRequestHeader requestHeader = new CreateTopicRequestHeader();
        requestHeader.setTopic(topicConfig.getTopicName());
        requestHeader.setDefaultTopic(defaultTopic);
        requestHeader.setReadQueueNums(topicConfig.getReadQueueNums());
        requestHeader.setWriteQueueNums(topicConfig.getWriteQueueNums());
        requestHeader.setPerm(topicConfig.getPerm());
        requestHeader.setTopicFilterType(topicConfig.getTopicFilterType().name());
        requestHeader.setTopicSysFlag(topicConfig.getTopicSysFlag());
        requestHeader.setOrder(topicConfig.isOrder());

        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.UPDATE_AND_CREATE_TOPIC, requestHeader);

        RemotingCommand response = this.remotingClient.invokeSync(MixAll.brokerVIPChannel(this.clientConfig.isVipChannelEnabled(), addr),
                request, timeoutMillis);
        assert response != null;
        switch (response.getCode()) {
            case ResponseCode.SUCCESS: {
                return;
            }
            default:
                break;
        }

        throw new MQClientException(response.getCode(), response.getRemark());
    }

以及其他的功能，如发送消息（Producer端需要的，也有上述对应的3种发送方式）、拉取消息（Consumer端需要的）等等

总结一下就是MQClientAPIImpl提供了很多的功能方法，如其名API，供调用者来使用。

## 3.3 服务的管理

目前MQClientAPIImpl提供了很多的一些API方法供我们使用，而MQClientInstance就是来具体使用这些API的，完成具体的服务的，这些服务有：

-	启动一些定时任务（定时更新Name Server列表、定时更新路由信息、持久化Consumer的offset等等）

-	启动拉取消息的线程服务（Consumer端需要的）

-	启动rebalance线程服务（Consumer端需要的）


所以不管是Producer还是Consumer，他们都会含有MQClientInstance。MQClientInstance目前就是一个大杂烩。

## 3.4 对用户暴漏的接口

MQProducer则是对用户暴漏的发送消息的相关接口，如上述案例中我们使用的接口方法，该接口的实现则是由上述三部分来支撑起来的。

# 4 同步和异步以及阻塞和非阻塞

在很多的通信编程过程中会面临2个选择，socket编程采用BIO还是NIO？对用户暴漏的接口方法采用同步还是异步？

通常socket采用的是NIO，即请求和响应要通过请求id来进行匹配

对用户暴漏的接口方法可以提供同步或者异步的方式，如果是同步的方式，即上述的ResponseFuture等待一个超时时间，则超时时间是需要的，如果是异步方式则不需要超时时间

如RocketMQ、ZooKeeper，他们都是采用NIO，在NIO的基础上对用户暴漏同步和异步接口方法。

# 5 Producer要关注的问题

来看下发送失败后重试问题：

## 5.1 同步发送

需要一个timeout时间的

我们知道同步发送的原理就是ResponseFuture等待一个timeout时间，如果超过该时间发送端认为发送失败，然后就会进行重试，目前RocketMQ的同步重试次数设置是

	private int retryTimesWhenSendFailed = 2;

默认是2

有一个重要问题就是：**在NIO基础上实现同步发送，发送超时重试会存在消息重复的问题**

同步发送等待一个timeout时间，如果超过该时间则发送端认为发送失败（但是服务器端可能已经接收并存储了该消息），此时客户端进行重试就可能会造成消息的重复。

其实在**在BIO基础上的同步发送，发送超时重试也会存在消息重复的问题**，即**超时重试都是可能存在重复问题的**。但是上述BIO和NIO超时的区别在于：如果是BIO超时则需要关闭该连接重新建立新的连接，如果是NIO超时则不需要关闭该连接。

重试都会从新选择消息队列来进行重试

## 5.2 异步发送

异步发送不需要超时时间

目前RocketMQ的重试配置如下：

	private int retryTimesWhenSendAsyncFailed = 2;

也默认是2次

重试都会从新选择消息队列来进行重试

## 5.3 顺序发送

顺序发送通过选择一个固定的消息队列来进行同步发送，目前RocketMQ的顺序发送没有重试次数。我感觉顺序发送也可以重试啊，只是不去重新选择消息队列就行了。