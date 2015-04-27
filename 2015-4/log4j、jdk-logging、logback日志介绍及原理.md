#1 需要解决的疑惑
目前的日志框架有jdk自带的logging，log4j1、log4j2、logback

目前用于实现日志统一的框架apache的commons-logging、slf4j

为了理清它们的关系，与繁杂的各种集成jar包，如下：

-	log4j、log4j-api、log4j-core
-	log4j-1.2-api、log4j-jcl、log4j-slf4j-impl、log4j-jul
-	logback-core、logback-classic、logback-access 
-	commons-logging
-	slf4j-api、slf4j-log4j12、slf4j-simple、jcl-over-slf4j、slf4j-jdk14、log4j-over-slf4j、slf4j-jcl

分成3篇文章来阐述

-	jdk自带的logging、log4j1、log4j2、logback的使用与原理简述
-	slf4j、apache的commons-logging与上述日志框架的集成原理
-	slf4j目前与commons-logging混用的境况


#2 jdk自带的logging

##2.1 使用案例

	private static final Logger logger=Logger.getLogger(JdkLoggingTest.class.getName());
	
	public static void main(String[] args){
		logger.info("jdk logging info: a msg");
	}

其中的Logger是：java.util.logging.Logger

##2.2 简单过程分析：

**不想看源码的请略过**

-	创建一个LogManager
	
	默认是java.util.logging.LogManager，但是也可以自定义，修改系统属性"java.util.logging.manager"即可，源码如下（manager就是LogManager）：

		try {
            cname = System.getProperty("java.util.logging.manager");
            if (cname != null) {
                try {
                    Class clz = ClassLoader.getSystemClassLoader().loadClass(cname);
                    manager = (LogManager) clz.newInstance();
                } catch (ClassNotFoundException ex) {
                    Class clz = Thread.currentThread().getContextClassLoader().loadClass(cname);
                    manager = (LogManager) clz.newInstance();
                }
            }
        } catch (Exception ex) {
            System.err.println("Could not load Logmanager \"" + cname + "\"");
            ex.printStackTrace();
        }
        if (manager == null) {
            manager = new LogManager();
        }

-	加载配置文件
	
	默认是jre目录下的lib/logging.properties文件，也可以自定义修改系统属性"java.util.logging.config.file",源码如下：

		String fname = System.getProperty("java.util.logging.config.file");
        if (fname == null) {
            fname = System.getProperty("java.home");
            if (fname == null) {
                throw new Error("Can't find java.home ??");
            }
            File f = new File(fname, "lib");
            f = new File(f, "logging.properties");
            fname = f.getCanonicalPath();
        }
        InputStream in = new FileInputStream(fname);
        BufferedInputStream bin = new BufferedInputStream(in);
        try {
            readConfiguration(bin);
		}

	
-	创建Logger，并缓存起来，放置到一个Hashtable中,并把LogManager设置进新创建的logger中

以tomcat为例，它就自定义了上述配置：

在tomcat的启动文件catalina.bat中，有如下设置：

-	修改属性"java.util.logging.manager",自定义LogManager，使用自己的ClassLoaderLogManager

		if not "%LOGGING_MANAGER%" == "" goto noJuliManager
		set LOGGING_MANAGER=-Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
		:noJuliManager
		set JAVA_OPTS=%JAVA_OPTS% %LOGGING_MANAGER%

-	修改属性"java.util.logging.config.file",自定义配置文件，使用自己的%CATALINA_BASE%\conf\logging.properties文件

		if not "%LOGGING_CONFIG%" == "" goto noJuliConfig
		set LOGGING_CONFIG=-Dnop
		if not exist "%CATALINA_BASE%\conf\logging.properties" goto noJuliConfig
		set LOGGING_CONFIG=-Djava.util.logging.config.file="%CATALINA_BASE%\conf\logging.properties"
		:noJuliConfig
		set JAVA_OPTS=%JAVA_OPTS% %LOGGING_CONFIG%

所以如果想研究tomcat的日志，可以从上面入手。

jdk自带的logging不再详细介绍，有兴趣的参见这篇文章[JDK Logging深入分析](http://blog.csdn.net/qingkangxu/article/details/7514770)

#3 log4j1

##3.1 使用案例

###3.1.1 需要的jar包

-	log4j

maven依赖如下：

	<dependency>
		<groupId>log4j</groupId>
		<artifactId>log4j</artifactId>
		<version>1.2.17</version>
	</dependency>

###3.1.2 使用方式

-	第一步：编写log4j.properties配置文件,放到类路径下
	
		log4j.rootLogger = debug, console
		log4j.appender.console = org.apache.log4j.ConsoleAppender
		log4j.appender.console.layout = org.apache.log4j.PatternLayout
		log4j.appender.console.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss} %m%n

	配置文件的详细内容不是本博客关注的重点，不再说明，自行搜索
	
-	第二步：代码中如下使用

		public class Log4jTest {
			private static final Logger logger=Logger.getLogger(Log4jTest.class);
			public static void main(String[] args){
				if(logger.isTraceEnabled()){
					logger.debug("log4j trace message");
				}
				if(logger.isDebugEnabled()){
					logger.debug("log4j debug message");
				}
				if(logger.isInfoEnabled()){
					logger.debug("log4j info message");
				}
			}
		}

-	补充： 
	
	-	1 上述方式默认到类路径下加载log4j.properties配置文件，如果log4j.properties配置文件不在类路径下，则可以选择如下方式之一来加载配置文件

		-	使用classLoader来加载资源	

			PropertyConfigurator.configure(Log4jTest.class.getClassLoader().getResource("properties/log4j.properties"));

		-	使用log4j自带的Loader来加载资源
			
			PropertyConfigurator.configure(Loader.getResource("properties/log4j.properties"));

##3.2 获取Logger的原理

**不想看源码的请略过**

本博客的重点不在于讲解log4j的架构。只是简单的说明获取一个Logger的过程。分三种情况来说明：

-	第一种情况：没有指定配置文件路径

	-	1 第一步： 引发LogManager的类初始化

		Logger.getLogger(Log4jTest.class)的源码如下：

			static public Logger getLogger(Class clazz) {
			    return LogManager.getLogger(clazz.getName());
			}

	-	2 第二步：初始化一个logger仓库Hierarchy

		Hierarchy的源码如下：
		
			public class Hierarchy implements LoggerRepository, RendererSupport, ThrowableRendererSupport {
			  	private LoggerFactory defaultFactory；
			  	Hashtable ht;
			  	Logger root;
				//其他略
			}
			
		-	LoggerFactory defaultFactory： 就是创建Logger的工厂
		-	Hashtable ht：用来存放上述工厂创建的Logger
		-	Logger root:作为根Logger

		LogManager在类初始化的时候如下方式来实例化Hierarchy：

			static {
    			Hierarchy h = new Hierarchy(new RootLogger((Level) Level.DEBUG));
				//略
			}

		new RootLogger作为root logger，默认是debug级别

		最后把Hierarchy绑定到LogManager上，可以在任何地方来获取这个logger仓库Hierarchy

	-	3 第三步：在LogManager的类初始化的过程中默认寻找类路径下的配置文件

		通过org.apache.log4j.helpers.Loader类来加载类路径下的配置文件：
		
			Loader.getResource("log4j.xml");
	  		Loader.getResource("log4j.properties")

		优先选择xml配置文件

	-	4 第四步：解析上述配置文件

		-	如果是xml文件则org.apache.log4j.xml.DOMConfigurator类来解析
		-	如果是properties文件，则使用org.apache.log4j.PropertyConfigurator来解析

		不再详细说明解析过程，看下解析后的结果：

		-	设置RootLogger的级别
		-	对RootLogger添加一系列我们配置的appender（我们通过logger来输出日志，通过logger中的appender指明了日志的输出目的地）
		
	-	5 第五步：当一切都准备妥当后，就该获取Logger了

		使用logger仓库Hierarchy中内置的LoggerFactory工厂来创建Logger了，并缓存起来，同时将logger仓库Hierarchy设置进新创建的Logger中
		
-	第二种情况，手动来加载不在类路径下的配置文件

	PropertyConfigurator.configure 执行时会去进行上述的配置文件解析，源码如下：

		public static void configure(java.net.URL configURL) {
			 new PropertyConfigurator().doConfigure(configURL,
			                    LogManager.getLoggerRepository());
		}

	-	仍然先会引发LogManager的类加载，创建出logger仓库Hierarchy，同时尝试加载类路径下的配置文件，此时没有则不进行解析，此时logger仓库Hierarchy中的RootLogger默认采用debug级别，没有appender而已。

	-	然后解析配置文件，对上述logger仓库Hierarchy的RootLogger进行级别的设置，添加appender

	-	此时再去调用Logger.getLogger，不会导致LogManager的类初始化（因为已经加载过了）

-	第三种情况，配置文件在类路径下，而我们又手动使用PropertyConfigurator去加载

	也就会造成2次加载解析配置文件，仅仅会造成覆盖而已（对于RootLogger进行从新设置级别，删除原有的appender，重新加载新的appender），所以多次加载解析配置文件以最后一次为准。

##3.3 主要对象总结

-	LogManager： 它的类加载会创建logger仓库Hierarchy，并尝试寻找类路径下的配置文件，如果有则解析

-	Hierarchy ： 包含三个重要属性：
				
	-	LoggerFactory logger的创建工厂
	-	Hashtable 用于存放上述工厂创建的logger
	-	Logger root logger,用于承载解析文件的结果，设置级别，同时存放appender

-	PropertyConfigurator: 用于解析log4j.properties文件

-	Logger : 我们用来输出日志的对象

#4 log4j2

##4.1 背景介绍
log4j2与log4j1发生了很大的变化，不兼容。log4j1仅仅作为一个实际的日志框架，slf4j、commons-logging作为门面，统一各种日志框架的混乱格局，现在log4j2也想跳出来充当门面了，也想统一大家了。哎，日志格局越来越混乱了。

log4j2分成2个部分：

-	log4j-api： 作为日志接口层，用于统一底层日志系统
-	log4j-core : 作为上述日志接口的实现，是一个实际的日志框架

##4.2 log4j2的使用案例

###4.2.1 需要的jar包

-	log4j-api
-	log4j-core

对应的maven依赖是：

	<dependency>
	    <groupId>org.apache.logging.log4j</groupId>
	    <artifactId>log4j-api</artifactId>
	    <version>2.2</version>
	</dependency>
  	<dependency>
    	<groupId>org.apache.logging.log4j</groupId>
    	<artifactId>log4j-core</artifactId>
    	<version>2.2</version>
  	</dependency>

###4.2.2 使用方式

-	第一步：编写log4j2.xml配置文件（目前log4j2只支持xml json yuml，不再支持properties文件）

		<?xml version="1.0" encoding="UTF-8"?>
		<Configuration status="WARN">
		  <Appenders>
		    <Console name="Console" target="SYSTEM_OUT">
		      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
		    </Console>
		  </Appenders>
		  <Loggers>
		    <Root level="debug">
		      <AppenderRef ref="Console"/>
		    </Root>
		  </Loggers>
		</Configuration>

-	第二步： 使用方式

		private static final Logger logger=LogManager.getLogger(Log4j2Test.class);
	
		public static void main(String[] args){
			if(logger.isTraceEnabled()){
				logger.debug("log4j trace message");
			}
			if(logger.isDebugEnabled()){
				logger.debug("log4j debug message");
			}
			if(logger.isInfoEnabled()){
				logger.debug("log4j info message");
			}
		}

和log4j1是不同的。此时Logger是log4j-api中定义的接口，而log4j1中的Logger则是类

##4.3 使用过程简单分析

**不想看源码的请略过**

-	获取底层使用的LoggerContextFactory：

	同样LogManager的类加载会去寻找log4j-api定义的LoggerContextFactory接口的底层实现，获取方式有三种：

	-	第一种： 尝试从jar中寻找log4j2.component.properties文件，如果配置了log4j2.loggerContextFactory则使用该LoggerContextFactory
	-	第二种：如果没找到，尝试从jar包中寻找META-INF/log4j-provider.properties文件，如log4j-core-2.2中就有该文件，如下图所示：
	![log4j-core提供的LoggerContextFactory][1]
	
	如果找到多个，取优先级最高的（该文件中指定了LoggerContextFactory，同时指定了优先级FactoryPriority），如log4j-core-2.2中log4j-provider.properties的文件内容如下：

		LoggerContextFactory = org.apache.logging.log4j.core.impl.Log4jContextFactory
		Log4jAPIVersion = 2.1.0
		FactoryPriority= 10

	-	第三种情况：上述方式还没找到，就使用默认的SimpleLoggerContextFactory

-	使用LoggerContextFactory获取LoggerContext

-	根据LoggerContext获取Logger

	以log4j-core为例：
	-	会首先判断LoggerContext是否被初始化过了，没有则进行初始化
	-	获取ConfigurationFactory,从配置中获取和插件中获取（log4j-core核心包中有三个YamlConfigurationFactory、JsonConfigurationFactory、XmlConfigurationFactory）
	-	以上文的案例中，会使用XmlConfigurationFactory来加载log4j2.xml配置文件
	-	LoggerContext初始化后，就可以获取或者创建Logger了


##4.4 主要对象总结

-	LogManager： 它的类加载会去寻找LoggerContextFactory接口的底层实现，会从jar包中的配置文件中寻找，如上面所述

-	LoggerContextFactory ： 用于创建LoggerContext，不同的日志实现系统会有不同的实现，如log4j-core中的实现为Log4jContextFactory
				
-	PropertyConfigurator: 用于解析log4j.properties文件

-	LoggerContext : 它包含了配置信息，并能创建log4j-api定义的Logger接口实例，并缓存这些实例

-	ConfigurationFactory：上述LoggerContext解析配置文件，需要用到ConfigurationFactory，目前有三个YamlConfigurationFactory、JsonConfigurationFactory、XmlConfigurationFactory，分别解析yuml json xml形式的配置文件

#5 logback

##5.1 使用案例

###5.1.1 需要的jar包

-	logback-core
-	logback-classic
-	slf4j-api

对应的maven依赖为：

	<dependency> 
    	<groupId>ch.qos.logback</groupId> 
    	<artifactId>logback-core</artifactId> 
    	<version>1.1.3</version> 
	</dependency> 
	<dependency> 
	    <groupId>ch.qos.logback</groupId> 
	    <artifactId>logback-classic</artifactId> 
	    <version>1.1.3</version> 
	</dependency>
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>slf4j-api</artifactId>
		<version>1.7.12</version>
	</dependency>

###5.1.2 使用方式

	private static final Logger logger=LoggerFactory.getLogger(LogbackTest.class);

	public static void main(String[] args){
		if(logger.isDebugEnabled()){
			logger.debug("slf4j-logback debug message");
		}
		if(logger.isInfoEnabled()){
			logger.debug("slf4j-logback info message");
		}
		if(logger.isTraceEnabled()){
			logger.debug("slf4j-logback trace message");
		}
		
		LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
	    StatusPrinter.print(lc);
	}

补充：

-	官方使用方式，其实就和slf4j集成了起来

	上述的Logger、LoggerFactory都是slf4j自己的接口与类

-	没有配置文件的情况下，使用的是默认配置。搜寻配置文件的过程如下：

	>1. Logback tries to find a file called **logback.groovy** in the classpath.

	>2. If no such file is found, logback tries to find a file called **logback-test.xml** in the classpath.

	>3. If no such file is found, it checks for the file **logback.xml** in the classpath..

	>4. If no such file is found, and the executing JVM has the ServiceLoader (JDK 6 and above) the ServiceLoader will be used to resolve an implementation of com.qos.logback.classic.spi.Configurator. The first implementation found will be used. See ServiceLoader documentation for more details.

	>5. If none of the above succeeds, logback configures itself automatically using the BasicConfigurator which will cause logging output to be directed to the console.

	>The fourth and last step is meant to provide a default (but very basic) logging functionality in the absence of a configuration file.

也可以在类路径下加上一个类似如下的logback.xml的配置文件，如下：

	<?xml version="1.0" encoding="UTF-8"?>
	<configuration>
	
	  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
	    <encoder>
	      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
	    </encoder>
	  </appender>
	
	  <root level="DEBUG">          
	    <appender-ref ref="STDOUT" />
	  </root>  
	  
	</configuration>

logback则会去解析对应的配置文件。

##5.3 使用过程简单分析

**不想看源码的请略过**

-	slf4j与底层的日志系统进行绑定

	在jar包中寻找org/slf4j/impl/StaticLoggerBinder.class 这个类，如在logback-classic中就含有这个类，如下图所示
	![logback与slf4j集成][2]

	如果找到多个StaticLoggerBinder，则表明目前底层有多个实际的日志框架，slf4j会随机选择一个

-	使用上述找到的StaticLoggerBinder创建一个实例，并返回一个ILoggerFactory实例：

		return StaticLoggerBinder.getSingleton().getLoggerFactory()；

	以logback-classic中的StaticLoggerBinder为例，在StaticLoggerBinder.getSingleton()过程中：会去加载解析配置文件
	源码如下：
		
		public URL findURLOfDefaultConfigurationFile(boolean updateStatus) {
		    ClassLoader myClassLoader = Loader.getClassLoaderOfObject(this);
			//寻找logback.configurationFile的系统属性
		    URL url = findConfigFileURLFromSystemProperties(myClassLoader, updateStatus);
		    if (url != null) {
		      return url;
		    }
			//寻找logback.groovy
		    url = getResource(GROOVY_AUTOCONFIG_FILE, myClassLoader, updateStatus);
		    if (url != null) {
		      return url;
		    }
			//寻找logback-test.xml
		    url = getResource(TEST_AUTOCONFIG_FILE, myClassLoader, updateStatus);
		    if (url != null) {
		      return url;
		    }
			//寻找logback.xml
		    return getResource(AUTOCONFIG_FILE, myClassLoader, updateStatus);
		}
	
	目前路径都是定死的，只有logback.configurationFile的系统属性是可以更改的，所以如果我们想更改配置文件的位置（不想放在类路径下），则需要设置这个系统属性：

		 System.setProperty("logback.configurationFile", "/path/to/config.xml");

	解析完配置文件后，返回的ILoggerFactory实例的类型是LoggerContext（它包含了配置信息）

-	根据返回的ILoggerFactory实例，来获取Logger

	就是根据上述的LoggerContext来创建一个Logger，每个logger与LoggerContext建立了关系，并放到LoggerContext的缓存中，就是LoggerContext的如下属性：
	
		private Map<String, Logger> loggerCache;

其实上述过程就是slf4j与其他日志系统的绑定过程。不同的日志系统与slf4j集成，都会有一个StaticLoggerBinder类，并会拥有一个ILoggerFactory的实现。
	

	
#6未完待续

这一篇文章简单介绍了几种日志的简单用法，下一篇文章就来介绍他们与commons-logging和slf4j怎么集成起来，就是要弄清楚各种集成jar包做了哪些事情

[1]: http://static.oschina.net/uploads/space/2015/0425/232556_MhSX_2287728.png
[2]: http://static.oschina.net/uploads/space/2015/0427/065133_Kqg8_2287728.png