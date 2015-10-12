#1 系列目录

-	[dubbo源码分析系列（1）扩展机制的实现](http://my.oschina.net/pingpangkuangmo/blog/508963)
-	[dubbo源码分析系列（2）服务的发布](http://my.oschina.net/pingpangkuangmo/blog/511766)
-	[dubbo源码分析系列（3）服务的引用](http://my.oschina.net/pingpangkuangmo/blog/515673)

#2 服务引用案例介绍

先看一个简单的客户端引用服务的例子，dubbo配置如下：

    <dubbo:application name="consumer-of-helloService" />
    
    <dubbo:registry  protocol="zookeeper"  address="127.0.0.1:2181" />
    
    <dubbo:reference id="helloService" interface="com.demo.dubbo.service.HelloService" />


-	使用zooKeeper作为注册中心
-	引用远程的HelloService接口服务

HelloService接口内容如下：

	public interface HelloService {
		public String hello(String msg);
	}


从[dubbo源码分析系列（2）服务的发布](http://my.oschina.net/pingpangkuangmo/blog/511766)这篇文章的前面部分就可以看到dubbo与Spring的接入过程的实质：

利用Spring的xml配置创建出一系列的配置对象，存至Spring容器中

-	application对应ApplicationConfig
-	registry对应RegistryConfig
-	monitor对应MonitorConfig
-	provider对应ProviderConfig
-	consumer对应ConsumerConfig
-	protocol对应ProtocolConfig
-	service对应ServiceConfig
-	reference对应ReferenceConfig

上面的对象不依赖Spring，也就是说你可以手动去创建上述对象。

为了在Spring启动的时候，也相应的启动provider发布服务注册服务的过程：又加入了一个和Spring相关联的ServiceBean，继承了ServiceConfig

为了在Spring启动的时候，也相应的启动consumer发现服务的过程：又加入了一个和Spring相关联的ReferenceBean，继承了ReferenceConfig

利用Spring就做了上述过程，得到相应的配置数据，然后启动相应的服务。如果想剥离Spring，我们就可以手动来创建上述配置对象，通过ServiceConfig和ReferenceConfig的API来启动相应的服务


具体针对上述案例，则是 根据dubbo:reference配置创建了一个ReferenceBean，该bean又实现了Spring的org.springframework.beans.factory.FactoryBean接口，所以我们如下方式使用时：

	@Autowired
    private HelloService helloService;

使用的不是ReferenceBean对象，而是ReferenceBean的getObject()方法返回的对象。该对象通过代理实现了HelloService接口。所以要看服务引用的整个过程就需要从ReferenceBean的getObject()方法开始入手。

下面来具体说明这个过程。

#3 服务引用过程

第一步：收集配置的参数，参数如下：

	methods=hello,
	timestamp=1443695417847,
	dubbo=2.5.3
	application=consumer-of-helloService
	side=consumer
	pid=7748
	interface=com.demo.dubbo.service.HelloService

第二步：从注册中心引用服务，创建出Invoker对象

如果是单个注册中心，代码如下：

	Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

	invoker = refprotocol.refer(interfaceClass, url);

上述url内容如下：

	registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?
	application=consumer-of-helloService&
	dubbo=2.5.3&
	pid=8292&
	registry=zookeeper&
	timestamp=1443707173909&

	refer=
		application=consumer-of-helloService&
		dubbo=2.5.3&
		interface=com.demo.dubbo.service.HelloService&
		methods=hello&
		pid=8292&
		side=consumer&
		timestamp=1443707173884&

前面的信息是注册中心的配置信息，如使用zookeeper来作为注册中心

后面refer的内容是要引用的服务信息，如引用HelloService服务

使用协议Protocol根据上述的url和服务接口来引用服务，创建出一个Invoker对象

第三步：使用ProxyFactory创建出一个接口的代理对象，该代理对象的方法的执行都交给上述Invoker来执行，代码如下：

	ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

	proxyFactory.getProxy(invoker);


下面就来详细的说明下上述第二步和第三步的过程中涉及到的几个概念

Protocol、Invoker、ProxyFactory

##3.1 概念介绍

分别介绍下Invoker、Protocol、ProxyFactory的概念

###3.1.1 Invoker概念

Invoker一个可执行对象。

这个概念已经在上一篇文章[dubbo源码分析系列（2）服务的发布](http://my.oschina.net/pingpangkuangmo/blog/511766#OSC_h3_9)中详细介绍了。这里再简单重复下

这个可执行对象的执行过程分成三种类型：

-	类型1：本地执行类的Invoker

-	类型2：远程通信执行类的Invoker

-	类型3：多个类型2的Invoker聚合成的集群版的Invoker

以HelloService接口方法为例：

-	本地执行类的Invoker： server端，含有对应的HelloServiceImpl实现，要执行该接口方法，仅仅只需要通过反射执行HelloServiceImpl对应的方法即可

-	远程通信执行类的Invoker： client端，要想执行该接口方法，需要需要进行远程通信，发送要执行的参数信息给server端，server端利用上述本地执行的Invoker执行相应的方法，然后将返回的结果发送给client端。这整个过程算是该类Invoker的典型的执行过程

-	集群版的Invoker：client端，拥有某个服务的多个Invoker，此时client端需要做的就是将这个多个Invoker聚合成一个集群版的Invoker，client端使用的时候，仅仅通过集群版的Invoker来进行操作。集群版的Invoker会从众多的远程通信类型的Invoker中选择一个来执行（从中加入路由和负载均衡策略），还可以采用一些失败转移策略等

所以来看下Invoker的实现情况：

![Invoker的实现情况](https://static.oschina.net/uploads/img/201509/27183301_Y1QA.png "Invoker的实现情况")

对于客户端来说，Invoker则应该是远程通信执行类的Invoker、多个远程通信类型的Invoker聚合成的集群版的Invoker这两种类型。先来说说非集群版的Invoker，即远程通信类型的Invoker。来看下DubboInvoker的具体实现

	protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
        
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
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
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

大概内容就是：

将通过远程通信将Invocation信息传递给服务器端，服务器端接收到该Invocation信息后，找到对应的本地Invoker，然后通过反射执行相应的方法，将方法的返回值再通过远程通信将结果传递给客户端。

这里分成3种情况：

-	执行的方法不需要返回值：直接使用ExchangeClient的send方法

-	执行的方法的结果需要异步返回：使用ExchangeClient的request方法，返回一个ResponseFuture，通过ThreadLocal方式与当前线程绑定，未等服务器端响应结果就直接返回

-	执行的方法的结果需要同步返回：使用ExchangeClient的request方法，返回一个ResponseFuture，一直阻塞到服务器端返回响应结果

###3.1.2 Protocol概念

从上面得知服务引用的第二个过程就是：

	invoker = refprotocol.refer(interfaceClass, url);

使用协议Protocol根据上述的url和服务接口来引用服务，创建出一个Invoker对象


针对server端来说，会如下使用Protocol

	Exporter<?> exporter = protocol.export(invoker);

Protocol要解决的问题就是：根据url中指定的协议（没有指定的话使用默认的dubbo协议）对外公布这个HelloService服务，当客户端根据协议调用这个服务时，将客户端传递过来的Invocation参数交给服务器端的Invoker来执行。所以Protocol加入了远程通信协议的这一块，根据客户端的请求来获取参数Invocation invocation。

而针对客户端，则需要根据服务器开放的协议（服务器端在注册中心注册的url地址中含有该信息）来创建相应的协议的Invoker对象，如

-	DubboInvoker
-	InjvmInvoker
-	ThriftInvoker

等等

如服务器端在注册中心中注册的url地址为：

	dubbo://192.168.1.104:20880/com.demo.dubbo.service.HelloService?
	anyhost=true&
	application=helloService-app&dubbo=2.5.3&
	interface=com.demo.dubbo.service.HelloService&
	methods=hello&
	pid=3904&
	side=provider&
	timestamp=1444003718316

会看到上述服务是以dubbo协议注册的，所以这里产生的Invoker就是DubboInvoker。我们来具体的看下这个过程

先来看下Protocol的接口定义：

	@Extension("dubbo")
	public interface Protocol {
	    
	    int getDefaultPort();

		//针对server端来说，将本地执行类的Invoker通过协议暴漏给外部。这样外部就可以通过协议发送执行参数Invocation，然后交给本地Invoker来执行
	    @Adaptive
		<T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
	
		//这个是针对客户端的，客户端从注册中心获取服务器端发布的服务信息
		//通过服务信息得知服务器端使用的协议，然后客户端仍然使用该协议构造一个Invoker。这个Invoker是远程通信类的Invoker。
		//执行时，需要将执行信息通过指定协议发送给服务器端，服务器端接收到参数Invocation，然后交给服务器端的本地Invoker来执行
	    @Adaptive
	    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
	
		void destroy();
	
	}


我们再来详细看看服务引用的第二步：

	invoker = refprotocol.refer(interfaceClass, url);

protocol的来历是：

	Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

我们从第一篇文章[dubbo源码分析系列（1）扩展机制的实现](http://my.oschina.net/pingpangkuangmo/blog/508963),可以知道上述获取Protocol protocol的原理，这里就不再多说了，直接贴出最终的Protocol的实现代码：

	public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException{
	    if (arg0 == null)  { 
	        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null"); 
	    }
	    if (arg0.getUrl() == null) { 
	        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null"); 
	    }
	    com.alibaba.dubbo.common.URL url = arg0.getUrl();
	    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
	    if(extName == null) {
	        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
	    }
	    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
	    return extension.export(arg0);
	}

	public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0,com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException{
	    if (arg1 == null)  { 
	        throw new IllegalArgumentException("url == null"); 
	    }
	    com.alibaba.dubbo.common.URL url = arg1;
	    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
	    if(extName == null) {
	        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
	    }
	    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
	    return extension.refer(arg0, arg1);
	}

refer(interfaceClass, url)的过程即根据url的配置信息来最终选择的Protocol实现，默认实现是"dubbo"的扩展实现即DubboProtocol，然后再对DubboProtocol进行依赖注入，进行wrap包装。先来看看Protocol的实现情况：

![Protocol的实现情况](https://static.oschina.net/uploads/img/201510/05083015_UsSq.png "Protocol的实现情况")

可以看到在返回DubboProtocol之前，经过了ProtocolFilterWrapper、ProtocolListenerWrapper、RegistryProtocol的包装。

所谓的包装就是如下类似的内容：

	package com.alibaba.xxx;
 
	import com.alibaba.dubbo.rpc.Protocol;
	 
	public class XxxProtocolWrapper implemenets Protocol {
	    Protocol impl;
	 
	    public XxxProtocol(Protocol protocol) { impl = protocol; }
	 
	    // 接口方法做一个操作后，再调用extension的方法
	    public Exporter<T> export(final Invoker<T> invoker) {
	        //... 一些操作
	        impl .export(invoker);
	        // ... 一些操作
	    }
	 
	    // ...
	}

使用装饰器模式，类似AOP的功能。

所以上述服务引用的过程

	invoker = refprotocol.refer(interfaceClass, urls.get(0));

中的refprotocol会先经过RegistryProtocol(先暂时忽略ProtocolFilterWrapper、ProtocolListenerWrapper)，它干了哪些事呢？

-	根据注册中心的registryUrl获取注册服务Registry，将自身的consumer信息注册到注册中心上

		//先根据客户端的注册中心配置找到对应注册服务
		Registry registry = registryFactory.getRegistry(url);

		//使用注册服务将客户端的信息注册到注册中心上
		registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                    Constants.CHECK_KEY, String.valueOf(false)));

	上述subscribeUrl地址如下：

		consumer://192.168.1.104/com.demo.dubbo.service.HelloService?
			application=consumer-of-helloService&
			dubbo=2.5.3&
			interface=com.demo.dubbo.service.HelloService&
			methods=hello&
			pid=6444&
			side=consumer&
			timestamp=1444606047076

	该url表述了自己是consumer，同时自己的ip地址是192.168.1.104，引用的服务是com.demo.dubbo.service.HelloService，以及注册时间等等

-	创建一个RegistryDirectory，从注册中心中订阅自己引用的服务，将订阅到的url在RegistryDirectory内部转换成Invoker

		RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
		directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
                Constants.PROVIDERS_CATEGORY 
                + "," + Constants.CONFIGURATORS_CATEGORY 
                + "," + Constants.ROUTERS_CATEGORY));

	上述RegistryDirectory是Directory的实现，Directory代表多个Invoker，可以把它看成List类型的Invoker，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更。

	RegistryDirectory内部含有两者重要属性：

	-	注册中心服务Registry registry
	-	Protocol protocol。

	它会利用注册中心服务Registry registry来获取最新的服务器端注册的url地址，然后再利用协议Protocol protocol将这些url地址转换成一个具有远程通信功能的Invoker对象，如DubboInvoker

	
-	然后使用Cluster cluster对象将上述多个Invoker对象（此时还没有真正创建出来，异步订阅，订阅成功之后，回调时才会创建出Invoker）聚合成一个集群版的Invoker对象。

		Cluster cluster = ExtensionLoader.getExtensionLoader(Cluster.class).getAdaptiveExtension();
	
		cluster.join(directory)
	
这里再详细看看Cluster接口：

	@SPI(FailoverCluster.NAME)
	public interface Cluster {
	
	    /**
	     * Merge the directory invokers to a virtual invoker.
	     * 
	     * @param <T>
	     * @param directory
	     * @return cluster invoker
	     * @throws RpcException
	     */
	    @Adaptive
	    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
	
	}

只有一个功能就是把上述Directory（相当于一个List类型的Invoker）聚合成一个Invoker，同时也可以对List进行过滤处理（这些过滤操作也是配置在注册中心的）等实现路由的功能，主要是对用户进行透明。看看接口实现情况：

![Cluster接口实现情况](https://static.oschina.net/uploads/img/201510/12080157_wsju.png "Cluster接口实现情况")


默认采用的是FailoverCluster，看下FailoverCluster：

	/**
	 * 失败转移，当出现失败，重试其它服务器，通常用于读操作，但重试会带来更长延迟。 
	 * 
	 * <a href="http://en.wikipedia.org/wiki/Failover">Failover</a>
	 * 
	 * @author william.liangf
	 */
	public class FailoverCluster implements Cluster {
	
	    public final static String NAME = "failover";
	
	    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
	        return new FailoverClusterInvoker<T>(directory);
	    }
	
	}

仅仅是创建了一个FailoverClusterInvoker，具体的逻辑留在调用的时候即调用该Invoker的invoke(final Invocation invocation)方法时来进行处理。其中又会涉及到另一个接口LoadBalance（从众多的Invoker中挑选出一个Invoker来执行此次调用任务），接口如下：

	@SPI(RandomLoadBalance.NAME)
	public interface LoadBalance {
	
		/**
		 * select one invoker in list.
		 * 
		 * @param invokers invokers.
		 * @param url refer url
		 * @param invocation invocation.
		 * @return selected invoker.
		 */
	    @Adaptive("loadbalance")
		<T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
	
	}

实现情况如下：

![LoadBalance接口实现情况](https://static.oschina.net/uploads/img/201510/12082143_bWmp.png "LoadBalance接口实现情况")

默认采用的是随机策略，具体的内容就请各自详细去研究。

###3.1.3 ProxyFactory概念

前一篇文章已经讲过了，对于server端，ProxyFactory主要负责将服务如HelloServiceImpl统一进行包装成一个Invoker，这些Invoker通过反射来执行具体的HelloServiceImpl对象的方法。而对于client端，则是将上述创建的集群版Invoker创建出代理对象。

接口定义如下：

	@Extension("javassist")
	public interface ProxyFactory {
	
	  	//针对client端，对Invoker对象创建出代理对象
	    @Adaptive({Constants.PROXY_KEY})
	    <T> T getProxy(Invoker<T> invoker) throws RpcException;
	
		//针对server端，将服务对象如HelloServiceImpl包装成一个Invoker对象
	    @Adaptive({Constants.PROXY_KEY})
	    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
	
	}

ProxyFactory的接口实现有JdkProxyFactory、JavassistProxyFactory，默认是JavassistProxyFactory，
JdkProxyFactory内容如下：

	public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
    }

可以看到是利用jdk自带的Proxy来动态代理目标对象Invoker。所以我们调用创建出来的代理对象如HelloService helloService的方法时，会执行InvokerInvocationHandler中的逻辑：

	public class InvokerInvocationHandler implements InvocationHandler {

	    private final Invoker<?> invoker;
	    
	    public InvokerInvocationHandler(Invoker<?> handler){
	        this.invoker = handler;
	    }
	
	    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	        String methodName = method.getName();
	        Class<?>[] parameterTypes = method.getParameterTypes();
	        if (method.getDeclaringClass() == Object.class) {
	            return method.invoke(invoker, args);
	        }
	        if ("toString".equals(methodName) && parameterTypes.length == 0) {
	            return invoker.toString();
	        }
	        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
	            return invoker.hashCode();
	        }
	        if ("equals".equals(methodName) && parameterTypes.length == 1) {
	            return invoker.equals(args[0]);
	        }
	        return invoker.invoke(new RpcInvocation(method, args)).recreate();
	    }
	
	}

可以看到还是交给目标对象Invoker来执行。

#4 结束语

本文简略地介绍了客户端引用服务过程以及涉及到的几个概念，接下来的打算是：

-	客户端与服务器端网络通信模块















