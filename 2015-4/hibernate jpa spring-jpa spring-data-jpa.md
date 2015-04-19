#1 需要解决的疑惑

目前jdbc、hibernate、jpa、spring之间有着千丝万缕的关系。在使用它们的时候有着各种各样的配置，初学者很容易分不清到底各自都做了什么事情，如果对自己要求高点，那就测试下下面几个问题：

-	jdbc的几个主要对象是什么？
-	jdbc的原生方式是什么样的？怎么配置？怎么使用事务？
-	hibernate的几个主要对象是什么？
-	hibernate的原生xml方式是什么样的？注解方式是什么样？怎么配置？怎么使用事务？
-	为什么会出现jpa?如果是你来设计，你该如何来设计jpa?同时事务又怎么办？
-	jpa的几个主要对象是什么？
-	jpa的原生的开发方式是什么样的？怎么配置？怎么使用事务？
-	spring为了让大家使用jpa更加方便，又做了哪些封装？又如何把datasource加进来？这时候，其他的orm又该如何集成？
-	spring为了再进一步的封装，产生spring-data-jpa，它的xml方式和注解方式又是什么样？它又是如何在前面的基础上做封装呢？

从上面可以再进一步总结：

-	事务的发展历程是什么样的？
-	对数据库的操作，xml方式和注解方式又是如何逐步演进的？

#2 实验案例

为了彻底搞清楚它们之间的关系，做了以下几个案例：

-	spring-jdbc：jdbc的原生方式开发和事务的使用

-	spring-jdbcTemplate：使用spring封装的JdbcTemplate开发和事务的使用

-	spring-hibernate-xml： hibernate的原生xml方式开发
	
-	spring-hibernate-xml-tx：hibernate的原生xml方式的事务的使用
	
-	spring-hibernate-annotation：hibernate的原生的注解方式开发
	
-	spring-hibernate-annotation-tx：hibernate的原生的注解方式的事务的使用
	
-	spring-jpa-hibernate：jpa的原生xml方式开发和事务的使用

-	spring-springjpa-hibernate:jpa的spring集成方式开发和事务的使用

-	spring-data-jpa-hibernate：spring-data-jpa方式开发和事务的使用

案例都是使用SpringMVC搭建的环境，主要是可以在此基础上方便开发。而学习过程的测试案例都是使用的是spring-test来进行测试的。

项目地址如下[jdbc-hibernate-jpa-spring](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring#hibernate-jpa-spring-三者的千丝万缕的关系)

#3 实验案例的详细说明：

##3.1数据库和表的准备

所有的案例都使用的数据库为test,表名为user,表user有id,name,age属性，User类如下：

	public class User {
		private Long id;
		private String name;
		private int age;
		//略
	}

目前使用的hibenrate版本是最新的4.3.8.Final。

##3.2 jdbc开发

###3.2.1 项目地址

该案例是项目[spring-jdbc](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring/tree/master/spring-jdbc)

###3.2.2 项目配置和使用分析

第一步： 先注册驱动类

	Class.forName("com.mysql.jdbc.Driver");

第二步： 使用DriverManager来获取连接Connection

	Connection conn=DriverManager.getConnection(url, user, password);

第三步： 使用Connection来执行sql语句，以及开启、回滚、提交事务

	Connection conn=jdbcDao.getConnection();
		conn.setAutoCommit(false);
		try {
			PreparedStatement ps=conn.prepareStatement("insert into user(name,age) value(?,?)");
			ps.setString(1,user.getName());
			ps.setInt(2,user.getAge());
			ps.execute();
			conn.commit();
		} catch (Exception e) {
			e.printStackTrace();
			conn.rollback();
		}finally{
			conn.close();
		}

缺点： 

-	每次都是获取一个连接，用完之后就释放掉，应该是用完再放回去，等待下一次使用，所以应该使用连接池datasource
-	Connection、PreparedStatement这种操作很麻烦，不方便
-	业务代码和事务代码相互交叉，都嵌套在try catch中，代码很不美观

针对上述问题，spring开发了JdbcTemplate来完成相应的封装

##3.3 spring-jdbcTemplate开发

###3.3.1 项目地址

该案例是项目[spring-jdbcTemplate](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring/tree/master/spring-jdbcTemplate)

###3.3.2 项目配置

首先注册dataSource,用于管理Connection，每次使用完Connection，仍旧放回连接池，等待下一次使用
	
	<bean id="dataSource"   
        class="com.mchange.v2.c3p0.ComboPooledDataSource"   
        destroy-method="close">   
        <property name="driverClass">   
            <value>${jdbc.driverClass}</value>   
        </property>   
        <property name="jdbcUrl">   
            <value>${jdbc.url}</value>   
        </property>   
        <property name="user">   
            <value>${jdbc.user}</value>   
        </property>   
        <property name="password">   
            <value>${jdbc.password}</value>   
        </property>   
        <!--连接池中保留的最小连接数。-->   
        <property name="minPoolSize">   
            <value>${jdbc.minPoolSize}</value>   
        </property>   
        <!--连接池中保留的最大连接数。Default: 15 -->   
        <property name="maxPoolSize">   
            <value>${jdbc.maxPoolSize}</value>   
        </property>   
        <!--初始化时获取的连接数，取值应在minPoolSize与maxPoolSize之间。Default: 3 -->   
        <property name="initialPoolSize">   
            <value>${jdbc.initialPoolSize}</value>   
        </property>   
    </bean>

然后注册jdbcTemplate，使用上述dataSource

	<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
		<property name="dataSource" ref="dataSource"></property>
	</bean>

对于事务，要注册对应的事务管理器，这里使用的是dataSource，所以要使用DataSourceTransactionManager，下面可以看到不同的事务管理器

	<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource"></property>
	</bean>

然后开启注解式的事务配置，即开启@Transactional的功能：

	<tx:annotation-driven proxy-target-class="true" transaction-manager="transactionManager"/>

###3.3.3 项目使用分析

配置说完了，来看具体的业务代码了：

	@Repository
	public class UserDao {
	
		@Autowired
		private JdbcTemplate jdbcTemplate;
		
		@Transactional
		public void save(User user){
			jdbcTemplate.update("insert into user(name,age) value(?,?)",user.getName(),user.getAge());
		}
	}

和之前的jdbc开发相比，就非常简洁了，一行代码就可以搞定。

来详细看下jdbcTemplate.update的执行过程，其实也是从dataSource中获取Connection，然后使用PreparedStatement这样的对象来完成对应的sql语句

源码如下：

	Connection con = DataSourceUtils.getConnection(getDataSource());
	PreparedStatement ps = null;
	try {
		Connection conToUse = con;
		if (this.nativeJdbcExtractor != null &&
				this.nativeJdbcExtractor.isNativeConnectionNecessaryForNativePreparedStatements()) {
			conToUse = this.nativeJdbcExtractor.getNativeConnection(con);
		}
		ps = psc.createPreparedStatement(conToUse);
		applyStatementSettings(ps);
		PreparedStatement psToUse = ps;
		if (this.nativeJdbcExtractor != null) {
			psToUse = this.nativeJdbcExtractor.getNativePreparedStatement(ps);
		}
		T result = action.doInPreparedStatement(psToUse);
		handleWarnings(ps);
		return result;
	}

我们再来看看，它又是如何来实现事务功能的。首先我们知道jdbcTemplate原理是就是对jdbc的封装，要想实现事务的功能，也必须对Connection执行以下的事务代码：

	conn.commit();
	conn.rollback();	

我们还知道一个重要信息就是，jdbc中执行sql的Connection和执行事务功能的的Connection是同一个Connection,这是非常需要明确的一点。

而Spring的@Transactional背后，则是使用SpringAOP来对我们的UserDao进行了代理。代理的方式有2种，jdk代理或者cglib代理。
如UserDao的实例使我们要代理的对象，暂且称作target

-	jdk代理针对那些实现了相应的接口的类，创建出的代理对象也是实现了同样的接口，同时该代理对象包含一个target
-	cglib代理针对类都可以进行代理，创建出的代理对象是继承了target类，同时该代理对象内部包含一个target

上述的具体细节见我的博客[SpringAOP源码分析系列](http://my.oschina.net/pingpangkuangmo/blog?catalog=3268217)

而我们采用的事务注解配置是：

	<tx:annotation-driven proxy-target-class="true" transaction-manager="transactionManager"/>

中采用的是proxy-target-class="true" 表示我们要使用cglib来进行代理，所以我们在使用UserDao的时候其实是使用的代理对象，这个代理对象其实是继承了UserDao，同时内部包含一个UserDao的实例，如下所示：
![我们实际使用的UserDao][1]

这个代理对象加入了事务拦截器TransactionInterceptor,可以想象成在真正执行代理对象内部target的save方法时，代理对象已经在外部进行了一层try catch包裹，在执行save的过程中，一旦发现异常，就使用transactionManager来进行事务的回滚。

而我们使用的DataSourceTransactionManager对事物的回滚也是通过Connection的rollback方法来完成的，DataSourceTransactionManager的源码如下：

	@Override
	protected void doRollback(DefaultTransactionStatus status) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
		Connection con = txObject.getConnectionHolder().getConnection();
		if (status.isDebug()) {
			logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
		}
		try {
			con.rollback();
		}
		catch (SQLException ex) {
			throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
		}
	}

此时我们知道了jdbcTemplate.update方法使用Connection来完成sql功能，事务的提交和回滚也是通过Connection来完成的，只有这两个Connection是同一个Connection的时候，才能真正起到事务作用。jdbcTemplate.update方法使用Connection是通过DataSourceUtils来获取的Connection,这个Connection是和当前线程绑定的，事务的提交和回滚使用的Connection也是和当前线程绑定的，所以他们是同一个Connection。具体的验证比较麻烦，可自行查看源码

jdbcTemplate只是简化了开发，仍然使用手写sql的方式来开发，没有使用orm，接下来就看看orm框架hibernate

##3.4 hibernate的原生xml方式开发

###3.4.1 项目地址

该案例是项目[spring-hibernate-xml](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring/tree/master/spring-hibernate-xml)

###3.4.2 项目配置

hibernate的原生xml方式涉及到2种配置文件

-	hibernate.cfg.xml ： 用于配置数据库连接和方言等等
		
		<hibernate-configuration>
		    <session-factory name="hibernateSessionFactory">
		        <property name="hibernate.connection.driver_class">com.mysql.jdbc.Driver</property>
		        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/</property>
		        <property name="hibernate.connection.username">root</property>
		        <property name="hibernate.connection.password">ligang</property>
		        <property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
		        <property name="hibernate.show_sql">true</property>
		        <property name="hibernate.default_schema">test</property>
		        <mapping resource="hibernate/mapping/User.hbm.xml"/>
		    </session-factory>
		</hibernate-configuration>

-	User.hbm.xml：用于配置实体User和数据库user表之间的关系

		<hibernate-mapping>
			<class name="com.demo.entity.User" table="user">
				<id name="id" column="id" type="long">
					<generator class="identity"/>
				</id>
				<property name="name" column="name" type="string"/>
				<property name="age" column="age" type="int"/>
			</class>
		</hibernate-mapping>

然后就是怎么使用hibernate原生的api来加载这两个文件。

-	加载hibernate.cfg.xml

		Configuration config=new Configuration();  
        config.configure("hibernate/base/hibernate.cfg.xml");  
        sessionFactory=config.buildSessionFactory(); 

	其中的文件路径是classpath路径下的hibernate/base/hibernate.cfg.xml。

	如果使用的是

		config.configure(); 

	则默认加载的是classpath路径下的hibernate.cfg.xml文件
	

-	加载User.hbm.xml

	-	1	直接在hibernate.cfg.xml文件中引入该文件

		如上述hibernate.cfg.xml方式

	-	2	使用代码添加类

			config.addClass(User.class);

		这种方式会在User类所在路径下寻找 User.cfg.xml文件

	-	3	使用代码添加资源映射文件

			config.addResource("hibernate/mapping/User.cfg.xml");

		这种方式会在classpath路径下寻找hibernate/mapping/User.cfg.xml。

注意，写路径的时候不要再写classpath : 了。

###3.4.3 主要的对象

SessionFactory、Session、Transaction 这些都是hibernate的原生的对象

-	SessionFactory

	刚才加载hibernate.cfg.xml配置文件就产生了这样的一个session工厂，它负责创建session

-	Session
	
	简单理解起来就是：Session和jdbc中一个Connection差不多，用于执行一些sql

-	Transaction

	是hibernate自己的事务对象，通过session可以开启一个事务，使用如下：
	
		Transaction tx=null;
		try {
			tx=session.beginTransaction();  
	        session.save(user);  
	        tx.commit();  
		} catch (Exception e) {
			if(tx!=null){
				tx.rollback();
			}
		}finally{
			session.close();
		}

[1]: http://static.oschina.net/uploads/space/2015/0419/215253_knUQ_2287728.png