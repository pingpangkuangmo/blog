#log4j1
现在已经出了log4j2，这里先说明log4j1
##使用案例

###需要的jar包

-	log4j

maven依赖如下：

	<dependency>
		<groupId>log4j</groupId>
		<artifactId>log4j</artifactId>
		<version>1.2.17</version>
	</dependency>

###使用方式

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

##获取Logger的原理
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

	PropertyConfigurator.configure 执行时会去进行上述的配置问价解析，源码如下：

		public static void configure(java.net.URL configURL) {
			 new PropertyConfigurator().doConfigure(configURL,
			                    LogManager.getLoggerRepository());
		}

	-	仍然先会引发LogManager的类加载，创建出logger仓库Hierarchy，同时尝试加载类路径下的配置文件，此时没有则不进行解析，此时logger仓库Hierarchy中的RootLogger默认采用debug级别，没有appender而已。

	-	然后解析配置文件，对上述logger仓库Hierarchy的RootLogger进行级别的设置，添加appender

	-	此时再去调用Logger.getLogger，不会导致LogManager的类初始化（因为已经加载过了）

-	第三种情况，配置文件在类路径下，而我们又手动使用PropertyConfigurator去加载

	也就会造成2次加载解析配置文件，仅仅会造成覆盖而已（对于RootLogger进行从新设置级别，删除原有的appender，重新加载新的appender），所以多次加载解析配置文件以最后一次为准。

##主要对象总结

-	LogManager： 它的类加载会创建logger仓库Hierarchy，并尝试寻找类路径下的配置文件，如果有则解析

-	Hierarchy ： 包含三个重要属性：
				
	-	LoggerFactory logger的创建工厂
	-	Hashtable 用于存放上述工厂创建的logger
	-	Logger root logger,用于承载解析文件的结果，设置级别，同时存放appender

-	PropertyConfigurator: 用于解析log4j.properties文件

-	Logger : 我们用来输出日志的对象

#log4j与slf4j集成

slf4j不是一个实际的日志框架,而log4j logback这类才是实际的日志框架，为了解决各大实际的日志框架使用上的不同，slf4j定义了一层日志使用接口，这就是slf4j-api包的内容，log4j logback等要集成进来，必须实现slf4j定义的日志接口。如log4j集成进slf4j，则需要log4j实现slf4j定义的接口，这就是slf4j-log4j12 jar包的内容。

##使用案例

###需要的jar包

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

###使用方式

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

	-	1 配置文件同样可以随意放置，如上述log4j的补充说明
	-	2 记住两者方式的不同：
			
			slf4j:  Logger logger=LoggerFactory.getLogger(Log4jSlf4JTest.class);
			log4j:  Logger logger=Logger.getLogger(Log4jTest.class);

		slf4j的Logger是slf4j定义的接口，而log4j的Logger是类。LoggerFactory是slf4j自己的类

##使用原理

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
##主要对象总结



[1]: http://static.oschina.net/uploads/space/2015/0425/103113_ofMj_2287728.png
[2]: http://static.oschina.net/uploads/space/2015/0425/104224_tX0x_2287728.png
[3]: http://static.oschina.net/uploads/space/2015/0425/104809_U2j1_2287728.png