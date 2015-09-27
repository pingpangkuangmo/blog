#1 系列目录

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

#3 服务的发布与注册过程

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

这里面就涉及到几个大的概念。ProxyFactory、Invoker、Protocol、Exporter。下面来一一介绍

##3.3 概念介绍

分别介绍下Invoker、ProxyFactory、Protocol、Exporter的概念

###3.3.1 Invoker概念

Invoker： 一个可执行的对象，能够根据方法名称、参数得到相应的执行结果。接口内容简略如下：

	public interface Invoker<T> {

	    Class<T> getInterface();
	
	    URL getUrl();
	    
	    Result invoke(Invocation invocation) throws RpcException;
	
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


















