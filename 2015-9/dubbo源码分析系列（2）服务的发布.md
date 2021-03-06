#1 系列目录

-	[dubbo源码分析系列（1）扩展机制的实现](http://my.oschina.net/pingpangkuangmo/blog/508963)
-	[dubbo源码分析系列（2）服务的发布](http://my.oschina.net/pingpangkuangmo/blog/511766)
-	[dubbo源码分析系列（3）服务的引用](http://my.oschina.net/pingpangkuangmo/blog/515673)
-	[dubbo源码分析系列（4）dubbo通信设计](http://my.oschina.net/pingpangkuangmo/blog/521945)

#2 dubbo与spring接入

dubbo的官方文档也说明了，dubbo可以不依赖任何Spring。这一块日后再详细说明，目前先介绍dubbo与Spring的集成。与spring的集成是基于Spring的Schema扩展进行加载

##2.1 Spring对外留出的扩展

用过Spring就知道可以在xml文件中进行如下配置：

	<context:component-scan base-package="com.demo.dubbo.server.serviceimpl"/>

	<context:property-placeholder location="classpath:config.properties"/>

	<tx:annotation-driven transaction-manager="transactionManager"/>

Spring是如何来解析这些配置呢？如果我们想自己定义配置该如何做呢？

对于上述的xml配置，分成三个部分

-	命名空间namespace，如tx、context
-	元素element，如component-scan、property-placeholder、annotation-driven
-	属性attribute，如base-package、location、transaction-manager

Spring定义了两个接口，来分别解析上述内容：

-	NamespaceHandler：注册了一堆BeanDefinitionParser，利用他们来进行解析
-	BeanDefinitionParser: 用于解析每个element的内容

来看下具体的一个案例，就以Spring的context命名空间为例，对应的NamespaceHandler实现是ContextNamespaceHandler：

	public class ContextNamespaceHandler extends NamespaceHandlerSupport {

		@Override
		public void init() {
			registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
			registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
			registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
			registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
			registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
			registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
			registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
			registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
		}
	
	}

注册了一堆BeanDefinitionParser，如果我们想看"component-scan"是如何实现的，就可以去看对应的ComponentScanBeanDefinitionParser的源码了

如果自定义了NamespaceHandler，如何加入到Spring中呢？

Spring默认会在加载jar包下的 META-INF/spring.handlers文件下寻找NamespaceHandler，默认的Spring文件如下：

![Spring Handlers](https://static.oschina.net/uploads/img/201509/24085142_ChaI.png "Spring Handlers")

文件内容如下：

	http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
	http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
	http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
	http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
	http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler

相应的命名空间使用相应的NamespaceHandler

##2.2 dubbo的接入实现

dubbo就是自定义类型的，所以也要给出NamespaceHandler、BeanDefinitionParser。NamespaceHandler是DubboNamespaceHandler：

	public class DubboNamespaceHandler extends NamespaceHandlerSupport {

		static {
			Version.checkDuplicate(DubboNamespaceHandler.class);
		}
	
	    public void init() {
	        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
	        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
	        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
	        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
	        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
	        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
	        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
	        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
	    }
	
	}

给出的BeanDefinitionParser全部是DubboBeanDefinitionParser，如果我们想看看<dubbo:registry>是怎么解析的，就可以去看看DubboBeanDefinitionParser的源代码。

而dubbo的jar包下，存在着META-INF/spring.handlers文件，内容如下：

	http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler

具体解析过程就不再说明了。结果就是不同的配置分别转换成Spring容器中的一个bean对象。

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

#3 服务的发布过程

##3.1 案例介绍

从上面知道，利用Spring的解析收集到很多一些配置，然后将这些配置都存至ServiceConfig中，然后调用ServiceConfig的export()方法来进行服务的发布与注册

先看一个简单的服务端例子，dubbo配置如下：

	<dubbo:application name="helloService-app" />
   
   	<dubbo:registry  protocol="zookeeper"  address="127.0.0.1:2181"  />
   
   	<dubbo:service interface="com.demo.dubbo.service.HelloService" ref="helloService" />
   
   	<bean id="helloService" class="com.demo.dubbo.server.serviceimpl.HelloServiceImpl"/>


-	有一个服务接口，HelloService，以及它对应的实现类HelloServiceImpl
-	将HelloService标记为dubbo服务，使用HelloServiceImpl对象来提供具体的服务
-	使用zooKeeper作为注册中心

##3.2 服务发布过程

一个服务可以有多个注册中心、多个服务协议

多注册中心信息：

首选根据注册中心配置，即上述的ZooKeeper配置信息，将注册信息聚合在一个URL对象中，registryURLs内容如下：

	[registry://192.168.1.104:2181/com.alibaba.dubbo.registry.RegistryService?application=helloService-app&localhost=true&registry=zookeeper]

多协议信息：

由于上述我们没有配置任何协议信息，就会使用默认的dubbo协议，开放在20880端口，也就是在该端口，对外提供上述的HelloService服务，注册的协议信息也转化成一个URL对象，如下:

	dubbo://192.168.1.104:20880/com.demo.dubbo.service.HelloService?anyhost=true&application=helloService-app&dubbo=2.0.13&interface=com.demo.dubbo.service.HelloService&methods=hello&prompt=dubbo&revision=

依据注册中心信息和协议信息的组合起来，依次来进行服务的发布。整个过程伪代码如下：

	List<URL> registryURLs = loadRegistries();
    for (ProtocolConfig protocolConfig : protocols) {
		//根据每一个协议配置构建一个URL
		URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);		
		for (URL registryURL : registryURLs) {
            String providerURL = url.toFullString();
            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(RpcConstants.EXPORT_KEY, providerURL));
            Exporter<?> exporter = protocol.export(invoker);
        }
	}

所以服务发布过程大致分成两步：

-	第一步：通过ProxyFactory将HelloServiceImpl封装成一个Invoker
-	第二步：使用Protocol将invoker导出成一个Exporter

这里面就涉及到几个大的概念。ProxyFactory、Invoker、Protocol、Exporter。下面来一一介绍

##3.3 概念介绍

分别介绍下Invoker、ProxyFactory、Protocol、Exporter的概念

###3.3.1 Invoker概念

Invoker： 一个可执行的对象，能够根据方法名称、参数得到相应的执行结果。接口内容简略如下：

	public interface Invoker<T> {

	    Class<T> getInterface();
	
	    URL getUrl();
	    
	    Result invoke(Invocation invocation) throws RpcException;

		void destroy();
	
	}

而Invocation则包含了需要执行的方法、参数等信息，接口定义简略如下：

	public interface Invocation {
  
	    URL getUrl();
	    
		String getMethodName();
	
		Class<?>[] getParameterTypes();
	
		Object[] getArguments();
	
	}

目前其实现类只有一个RpcInvocation。内容大致如下：

	public class RpcInvocation implements Invocation, Serializable {
	
	    private String              methodName;
	
	    private Class<?>[]          parameterTypes;
	
	    private Object[]            arguments;
	
	    private transient URL       url;
	}

仅仅提供了Invocation所需要的参数而已，继续回到Invoker

这个可执行对象的执行过程分成三种类型：

-	类型1：本地执行类的Invoker

-	类型2：远程通信执行类的Invoker

-	类型3：多个类型2的Invoker聚合成的集群版的Invoker

以HelloService接口方法为例：

-	本地执行类的Invoker： server端，含有对应的HelloServiceImpl实现，要执行该接口方法，仅仅只需要通过反射执行HelloServiceImpl对应的方法即可

-	远程通信执行类的Invoker： client端，要想执行该接口方法，需要需要进行远程通信，发送要执行的参数信息给server端，server端利用上述本地执行的Invoker执行相应的方法，然后将返回的结果发送给client端。这整个过程算是该类Invoker的典型的执行过程

-	集群版的Invoker：client端，拥有某个服务的多个Invoker，此时client端需要做的就是将这个多个Invoker聚合成一个集群版的Invoker，client端使用的时候，仅仅通过集群版的Invoker来进行操作。集群版的Invoker会从众多的远程通信类型的Invoker中选择一个来执行（从中加入负载均衡策略），还可以采用一些失败转移策略等

所以来看下Invoker的实现情况：

![Invoker的实现情况](https://static.oschina.net/uploads/img/201509/27183301_Y1QA.png "Invoker的实现情况")

###3.3.2 ProxyFactory概念

对于server端，主要负责将服务如HelloServiceImpl统一进行包装成一个Invoker，这些Invoker通过反射来执行具体的HelloServiceImpl对象的方法。

接口定义如下：

	@Extension("javassist")
	public interface ProxyFactory {
	
	  	//针对client端，创建出代理对象
	    @Adaptive({Constants.PROXY_KEY})
	    <T> T getProxy(Invoker<T> invoker) throws RpcException;
	
		//针对server端，将服务对象如HelloServiceImpl包装成一个Invoker对象
	    @Adaptive({Constants.PROXY_KEY})
	    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
	
	}

ProxyFactory的接口实现有JdkProxyFactory、JavassistProxyFactory，默认是JavassistProxyFactory，
JdkProxyFactory内容如下：

	public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, 
                                      Class<?>[] parameterTypes, 
                                      Object[] arguments) throws Throwable {
                Method method = proxy.getClass().getMethod(methodName, parameterTypes);
                return method.invoke(proxy, arguments);
            }
        };
    }

可以看到是创建了一个AbstractProxyInvoker（这类就是本地执行的Invoker），它对Invoker的Result invoke(Invocation invocation)实现如下：

	public Result invoke(Invocation invocation) throws RpcException {
        try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy " + invocation + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

综上所述，服务发布的第一个过程就是：

使用ProxyFactory将HelloServiceImpl封装成一个本地执行的Invoker。


###3.3.3 Protocol概念

从上面得知服务发布的第一个过程就是：

使用ProxyFactory将HelloServiceImpl封装成一个本地执行的Invoker。

执行这个服务，即执行这个本地Invoker，即调用这个本地Invoker的invoke(Invocation invocation)方法，方法的执行过程就是通过反射执行了HelloServiceImpl的内容。现在的问题是：这个方法的参数Invocation invocation的来源问题。


针对server端来说，Protocol要解决的问题就是：根据指定协议对外公布这个HelloService服务，当客户端根据协议调用这个服务时，将客户端传递过来的Invocation参数交给上述的Invoker来执行。所以Protocol加入了远程通信协议的这一块，根据客户端的请求来获取参数Invocation invocation。

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


我们再来详细看看服务发布的第二步：

	Exporter<?> exporter = protocol.export(invoker);

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

export(Invoker invoker)的过程即根据Invoker中url的配置信息来最终选择的Protocol实现，默认实现是"dubbo"的扩展实现即DubboProtocol，然后再对DubboProtocol进行依赖注入，进行wrap包装。先来看看Protocol的实现情况：

![Protocol的实现情况](https://static.oschina.net/uploads/img/201509/27223034_cH5E.png "Protocol的实现情况")

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

下面主要讲解RegistryProtocol和DubboProtocol，先暂时忽略ProtocolFilterWrapper、ProtocolListenerWrapper

所以上述服务发布的过程

	Exporter<?> exporter = protocol.export(invoker)；

会先经过RegistryProtocol，它干了哪些事呢？

-	利用内部的Protocol即DubboProtocol，将服务进行导出，如下

		exporter = protocol.export(new InvokerWrapper<T>(invoker, url));

-	根据注册中心的registryUrl获取注册服务Registry，然后将serviceUrl注册到注册中心上,供客户端订阅

		Registry registry = registryFactory.getRegistry(registryUrl);
        registry.register(serviceUrl)


来详细看看上述DubboProtocol的服务导出功能：

-	首先根据Invoker的url获取ExchangeServer通信对象（负责与客户端的通信模块），以url中的host和port作为key存至Map<String, ExchangeServer> serverMap中。即可以采用全部服务的通信交给这一个ExchangeServer通信对象，也可以某些服务单独使用新的ExchangeServer通信对象。

		String key = url.getAddress();
        //client 也可以暴露一个只有server可以调用的服务。
        boolean isServer = url.getParameter(RpcConstants.IS_SERVER_KEY,true);
        if (isServer && ! serverMap.containsKey(key)) {
            serverMap.put(key, getServer(url));
        }

-	创建一个DubboExporter，封装invoker。然后根据url的port、path（接口的名称）、版本号、分组号作为key，将DubboExporter存至Map<String, Exporter<?>> exporterMap中

		key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);


现在我们要搞清楚我们的目的：通过通信对象获取客户端传来的Invocation invocation参数，然后找到对应的DubboExporter（即能够获取到本地Invoker）就可以执行服务了。

上述每一个ExchangeServer通信对象都绑定了一个ExchangeHandler requestHandler对象，内容简略如下：

	private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
        
        public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                Invocation inv = (Invocation) message;
                Invoker<?> invoker = getInvoker(channel, inv);
                RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
                return invoker.invoke(inv);
            }
            throw new RemotingException(channel, "Unsupported request: " + message == null ? null : (message.getClass().getName() + ": " + message) + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
        }
    };

可以看到在获取到Invocation参数后，调用getInvoker(channel, inv)来获取本地Invoker。获取过程就是根据channel获取port，根据Invocation inv信息获取要调用的服务接口、版本号、分组号等，以此组装成key，从上述Map<String, Exporter<?>> exporterMap中获取Exporter，然后就可以找到对应的Invoker了，就可以顺利的调用服务了。

而对于通信这一块，接下来会专门来详细的说明。

###3.3.4 Exporter概念

负责维护invoker的生命周期。接口定义如下：

	public interface Exporter<T> {
  
	    Invoker<T> getInvoker();
	
	    void unexport();
	
	}

包含了一个Invoker对象。一旦想撤销该服务，就会调用Invoker的destroy()方法，同时清理上述exporterMap中的数据。对于RegistryProtocol来说就需要向注册中心撤销该服务。


#4 结束语

本文简略地介绍了接入Spring过程的原理，以及服务发布过程中的几个概念。接下来的打算是：

-	客户端订阅服务与使用服务涉及的概念
-	注册中心模块
-	客户端与服务器端网络通信模块















