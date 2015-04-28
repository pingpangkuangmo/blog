#1 系列目录

-	[jdk-logging、log4j、logback日志介绍及原理](http://my.oschina.net/pingpangkuangmo/blog/406618)
-	[commons-logging与jdk-logging、log4j1、log4j2、logback的集成](http://my.oschina.net/pingpangkuangmo/blog/406618)

#2 slf4j

先从一个简单的使用案例来说明

##2.1 简单的使用案例

	private static Logger logger=LoggerFactory.getLogger(Log4jSlf4JTest.class);
	
	public static void main(String[] args){
		if(logger.isDebugEnabled()){
			logger.debug("slf4j-log4j debug message");
		}
		if(logger.isInfoEnabled()){
			logger.debug("slf4j-log4j info message");
		}
		if(logger.isTraceEnabled()){
			logger.debug("slf4j-log4j trace message");
		}
	}

上述Logger接口、LoggerFactory类都是slf4j自己定义的。

##2.2 使用原理

LoggerFactory.getLogger(Log4jSlf4JTest.class)的源码如下：

	public static Logger getLogger(String name) {
        ILoggerFactory iLoggerFactory = getILoggerFactory();
        return iLoggerFactory.getLogger(name);
    }

上述获取Log的过程大致分成2个阶段

-	获取ILoggerFactory的过程 (从字面上理解就是生产Logger的工厂)
-	根据ILoggerFactory获取Logger的过程

下面来详细说明：

-	1 获取ILoggerFactory的过程

	又可以分成3个过程：
	
	-	1.1 从类路径中寻找org/slf4j/impl/StaticLoggerBinder.class类

			ClassLoader.getSystemResources("org/slf4j/impl/StaticLoggerBinder.class")
		
	-	1.2 随机选取一个StaticLoggerBinder.class来创建一个单例

			StaticLoggerBinder.getSingleton()
	
	-	1.3 根据上述创建的StaticLoggerBinder单例，返回一个ILoggerFactory实例

			StaticLoggerBinder.getSingleton().getLoggerFactory()

	所以slf4j与其他实际的日志框架的集成jar包中，都会含有这样的一个org/slf4j/impl/StaticLoggerBinder.class类文件

-	2 根据ILoggerFactory获取Logger的过程

	这就要看具体的ILoggerFactory类型了，下面的集成来详细说明

#3 slf4j与jdk-logging集成

##3.1 需要的jar包

-	slf4j-api 
-	slf4j-jdk14

对应的maven依赖为：

	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>slf4j-jdk14</artifactId>
		<version>1.7.12</version>
	</dependency>

##3.2 使用案例

	private static final Logger logger=LoggerFactory.getLogger(JulSlf4jTest.class);
	
	public static void main(String[] args){
		if(logger.isDebugEnabled()){
			logger.debug("jul debug message");
		}
		if(logger.isInfoEnabled()){
			logger.info("jul info message");
		}
		if(logger.isWarnEnabled()){
			logger.warn("jul warn message");
		}
	}

上述的Logger、LoggerFactory都是slf4j自己的API中的内容，没有jdk自带的logging的踪影，然后打出来的日志却是通过jdk自带的logging来输出的，如下：

	四月 28, 2015 7:33:20 下午 com.demo.log4j.JulSlf4jTest main
	信息: jul info message
	四月 28, 2015 7:33:20 下午 com.demo.log4j.JulSlf4jTest main
	警告: jul warn message

##3.3 使用原理分析

先看下slf4j-jdk14 jar包中的内容：

![jul与slf4j集成][4]

从中可以看到：

-	的确是有org/slf4j/impl/StaticLoggerBinder.class类
-	该StaticLoggerBinder返回的ILoggerFactory类型将会是JDK14LoggerFactory
-	JDK14LoggerAdapter就是实现了slf4j定义的Logger接口

下面梳理下真个流程：

-	1 获取ILoggerFactory的过程

	由于类路径下有org/slf4j/impl/StaticLoggerBinder.class，所以会选择slf4j-jdk14中的StaticLoggerBinder来创建单例对象并返回ILoggerFactory，来看下StaticLoggerBinder中的ILoggerFactory是什么类型：

		private StaticLoggerBinder() {
	        loggerFactory = new org.slf4j.impl.JDK14LoggerFactory();
	    }
	
	所以返回了JDK14LoggerFactory的实例

-	2 根据ILoggerFactory获取Logger的过程

	来看下JDK14LoggerFactory是如何返回一个slf4j定义的Logger接口的实例的，源码如下：

		java.util.logging.Logger julLogger = java.util.logging.Logger.getLogger(name);
        Logger newInstance = new JDK14LoggerAdapter(julLogger);

	可以看到，就是使用jdk自带的logging的原生方式来先创建一个jdk自己的java.util.logging.Logger实例

	然后利用JDK14LoggerAdapter将上述的java.util.logging.Logger包装成slf4j定义的Logger实例

	所以我们使用slf4j来进行编程，最终会委托给jdk自带的java.util.logging.Logger去执行。

#4 slf4j与log4j1集成

##4.1 需要的jar包

-	slf4j-api  
-	slf4j-log4j12
-	log4j

maven依赖分别为：

	<!-- slf4j -->
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>slf4j-api</artifactId>
		<version>1.7.12</version>
	</dependency>
	
	<!-- slf4j-log4j -->
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>slf4j-log4j12</artifactId>
		<version>1.7.12</version>
	</dependency>

	<!-- log4j -->
    <dependency>
		<groupId>log4j</groupId>
		<artifactId>log4j</artifactId>
		<version>1.2.17</version>
	</dependency>

##4.2 使用案例

-	第一步：编写log4j.properties配置文件,放到类路径下

		log4j.rootLogger = debug, console
		log4j.appender.console = org.apache.log4j.ConsoleAppender
		log4j.appender.console.layout = org.apache.log4j.PatternLayout
		log4j.appender.console.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss} %m%n

	配置文件的详细内容不是本博客关注的重点，不再说明，自行搜索

-	第二步：代码中如下使用

		private static Logger logger=LoggerFactory.getLogger(Log4jSlf4JTest.class);
	
		public static void main(String[] args){
			if(logger.isDebugEnabled()){
				logger.debug("slf4j-log4j debug message");
			}
			if(logger.isInfoEnabled()){
				logger.debug("slf4j-log4j info message");
			}
			if(logger.isTraceEnabled()){
				logger.debug("slf4j-log4j trace message");
			}
		}

-	补充说明：

	-	1 配置文件同样可以随意放置，如log4j原生方式加载配置文件的方式[log4j原生开发](http://my.oschina.net/pingpangkuangmo/blog/406618#OSC_h1_5)
	-	2 记住两者方式的不同：
			
			slf4j:  Logger logger=LoggerFactory.getLogger(Log4jSlf4JTest.class);
			log4j:  Logger logger=Logger.getLogger(Log4jTest.class);

		slf4j的Logger是slf4j定义的接口，而log4j的Logger是类。LoggerFactory是slf4j自己的类

##4.3 使用案例分析

就需要详细查看LoggerFactory.getLogger的源码了，如下：

	public static Logger getLogger(String name) {
        ILoggerFactory iLoggerFactory = getILoggerFactory();
        return iLoggerFactory.getLogger(name);
    }

分成2大步：

-	获取对应的ILoggerFactory
-	使用ILoggerFactory来生产Logger

来看下具体过程：

-	获取对应的ILoggerFactory

	ILoggerFactory： 是slf4j定义的logger工厂接口，其他日志框架要与sl4fj集成起来。必须实现这个接口，如log4j要集成进来，对应的实现类是Log4jLoggerFactory。

	ILoggerFactory是由StaticLoggerBinder来创建出来的

	ILoggerFactory的获取过程如下：

	-	第一个过程：slf4j寻找绑定类StaticLoggerBinder

		使用ClassLoader来加载 "org/slf4j/impl/StaticLoggerBinder.class"这样的类的url
		
		如果找到多个，则输出 Class path contains multiple SLF4J bindings 表示有多个日志实现与slf4j进行了绑定

		我们可以从log4j与sf4j中集成jar包slf4j-log4j12看到有org/slf4j/impl/StaticLoggerBinder这样的类，见下图
		
		![log4j与slf4j的集成][1]

		我们也可以从logback与slf4j中集成中也可以看,logback没有把集成包像slf4j-log4j12一样单独出来，而是把集成的内容嵌到logback-classic中去了，如下图所示：
		
		![logback与slf4j的集成][2]

		我们也可以从jdk1.4自带的logging与slf4j集成的jar包slf4j-jdk14中可以看出，也有org/slf4j/impl/StaticLoggerBinder这个类，如下图：

		![jdk1.4自带的logging与slf4j的集成][3]

		现在问题是，当你的类路径下有多个StaticLoggerBinder的时候，slf4j会随机选择一个，见[slf4j的官方说明](http://www.slf4j.org/codes.html#multiple_bindings),如下：

		>The warning emitted by SLF4J is just that, a warning. Even when multiple bindings are present, SLF4J will pick one logging framework/implementation and bind with it. The way SLF4J picks a binding is determined by the JVM and for all practical purposes should be considered random

		下面看下当出现多个StaticLoggerBinder的时候的输出日志（简化了一些内容）：
		
			SLF4J: Class path contains multiple SLF4J bindings.
			SLF4J: Found binding in [slf4j-log4j12-1.7.12.jar!/org/slf4j/impl/StaticLoggerBinder.class]
			SLF4J: Found binding in [logback-classic-1.1.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
			SLF4J: Found binding in [slf4j-jdk14-1.7.12.jar!/org/slf4j/impl/StaticLoggerBinder.class]
			SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
			SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
		
	-	第二个过程：创建出StaticLoggerBinder实例，并创建出ILoggerFactory

		源码如下：
			
			StaticLoggerBinder.getSingleton().getLoggerFactory()

		以slf4j-log4j12中的StaticLoggerBinder为例，创建出的ILoggerFactory为Log4jLoggerFactory



[1]: http://static.oschina.net/uploads/space/2015/0425/103113_ofMj_2287728.png
[2]: http://static.oschina.net/uploads/space/2015/0425/104224_tX0x_2287728.png
[3]: http://static.oschina.net/uploads/space/2015/0425/104809_U2j1_2287728.png

jul-slf4j的集成
[4]: http://static.oschina.net/uploads/space/2015/0428/193549_XRSY_2287728.png
