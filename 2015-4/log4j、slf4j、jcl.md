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

		-	PropertyConfigurator.configure(Log4jTest.class.getClassLoader().getResource("properties/log4j.properties"));
		-	PropertyConfigurator.configure(Loader.getResource("properties/log4j.properties"));

##获取Logger的原理
本博客的重点不在于讲解log4j的架构。只是简单的说明获取一个Logger的过程。分两种情况来说明：

-	没有指定配置文件路径

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

		Logger root则为上述的new RootLogger，默认是debug级别

		最后把Hierarchy绑定到LogManager上，可以在任何地方来获取这个logger仓库Hierarchy

	-	3 第三步：在LogManager的类初始化的过程中默认寻找类路径下的配置文件

		通过org.apache.log4j.helpers.Loader类来加载类路径下的配置文件：
		
			Loader.getResource("log4j.xml");
	  		Loader.getResource("log4j.properties")

		优先选择xml配置文件

	-	4 第四步：解析上述配置文件

		-	如果是xml文件则org.apache.log4j.xml.DOMConfigurator类来解析
		-	如果是properties文件，则使用org.apache.log4j.PropertyConfigurator来解析

		
			
		
-	使用PropertyConfigurator来加载配置文件


###主要对象

-	Logger： org.apache.log4j.Logger