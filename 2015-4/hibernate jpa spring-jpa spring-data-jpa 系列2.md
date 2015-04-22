#1 前面的文章索引

-	[jdbc开发和事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_5)
-	[spring-jdbcTemplate开发和事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_8)
-	[hibernate的原生xml方式开发和事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_12)
-	[hibernate的原生xml方式与spring集成以及事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_17)
-	[hibernate的原生注解方式开发和事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_21)
-	[hibernate的注解方式开发与Spring集成和事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_24)

详见第一篇文章[jdbc-jdbcTemplate-hibernate-jpa-springDataJpa系列（一）](http://my.oschina.net/pingpangkuangmo/blog/404280)

这里继续第一篇文章的内容，开始介绍jpa

#2 jpa原生开发和事务的使用

##2.1 jpa的来源

由于各种orm框架层出不穷,为了统一大家，就出现了jpa这一层接口规范，不同的orm框架都去实现这一规范，然后我们就只关心使用jpa的编程接口来进行编程，不用再关系底层到底使用的是那种orm框架，同时也很容易切换底层所使用的orm框架

##2.2 项目地址

-	[spring-jpa-hibernate](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring/tree/master/spring-jpa-hibernate)

##2.3 jpa的配置

-	第一步：就是jpa的核心配置文件 persistence.xml

	默认情况下该配置文件存放的位置是classpath路径下 META-INF/persistence.xml。配置文件的内容如下：

		<persistence-unit name="test" transaction-type="RESOURCE_LOCAL">
	        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
	        <properties>
	            <property name="hibernate.connection.driver_class" value="com.mysql.jdbc.Driver" />
	            <property name="hibernate.connection.url" value="jdbc:mysql://localhost:3306/test" />
	            <property name="hibernate.connection.username" value="root" />
	            <property name="hibernate.connection.password" value="ligang" />
	            <property name="hibernate.show_sql" value="true" />
	            <property name="hibernate.hbm2ddl.auto" value="update" />
	        </properties> 
	    </persistence-unit>

	我们可以看到这个核心配置文件其实就是指明了jpa底层所使用的是何种orm框架，这里称作为provider。然后properties里面的内容，其实都是为provider准备的一些配置信息。

-	第二步: 根据核心配置文件创建EntityManagerFactory

		EntityManagerFactory entityManagerFactory=Persistence.createEntityManagerFactory("test");

-	第三步：根据EntityManagerFactory创建出EntityManager

		EntityManager entityManager=entityManagerFactory.createEntityManager();

-	第四步：根据EntityManager对实体entity进行增删改查

		entityManager.persist(user)

##2.4 jpa的几个重要对象说明

**建议与Hibenrate的几个重要的原生对象进行对比，地址**[Hibernate的原生xml方式开发和事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_12)

-	PersistenceProvider接口： 

	根据持久化单元名称和配置参数创建出EntityManagerFactory，接口定义如下：

		public interface PersistenceProvider {
			public EntityManagerFactory createEntityManagerFactory(String emName, Map map);
			public EntityManagerFactory createContainerEntityManagerFactory(PersistenceUnitInfo info, Map map);
			//略
		}

	jpa仅仅是一层接口规范，不同的底层的实现者提供各自的provider。如hibernate提供的provider实现是org.hibernate.jpa.HibernatePersistenceProvider

-	EntityManagerFactory接口：

	就是EntityManager的工厂。其实可以类似于Hibernate中的SessionFactory。对于Hibernate来说，其实它就是对SessionFactory的封装，Hibernate实现的EntityManagerFactory是EntityManagerFactoryImpl，源码如下：

		public class EntityManagerFactoryImpl implements HibernateEntityManagerFactory {
		
			private final transient SessionFactoryImpl sessionFactory;
			//略
		}
		
-	EntityManager接口：

	能够对实体entity进行增删改查，其实可以类似于Hibernate中的Session。对于Hibernate来说，其实它就是对Session的封装。Hibernate实现的EntityManager是EntityManagerImpl，源码如下：

		public class EntityManagerImpl extends AbstractEntityManagerImpl implements SessionOwner {

			protected Session session;
			//略
		}

-	EntityTransaction接口：

	jpa定义的事务接口，其实可以类似于Hibernate原生的的Transaction接口。对于Hibernate来说，其实它就是对Transaction的封装。Hibernate实现的EntityTransaction是TransactionImpl，源码如下：

		public class TransactionImpl implements EntityTransaction {

			private Transaction tx;
			//略
		}

##2.5 jpa的使用案例

	@Repository
	public class JpaDao {
		private EntityManagerFactory entityManagerFactory;
		public JpaDao(){
			entityManagerFactory=Persistence.createEntityManagerFactory("test");
		}
		public EntityManager getEntityManager(){
			return entityManagerFactory.createEntityManager();
		}
	}

	@Repository
	public class UserDao {
		@Autowired
		private JpaDao jpaDao;
		public void save(User user){
			EntityManager entityManager=jpaDao.getEntityManager();
			EntityTransaction tx=null;
			try {
				tx=entityManager.getTransaction();
				tx.begin();
				entityManager.persist(user);
				tx.commit();
			} catch (Exception e) {
				if(tx!=null){
					tx.rollback();
				}
			}finally{
				entityManager.close();
			}
		}
	}

我们可以看到，上述的save过程和Hibernate的过程非常相似，只不过把Hibernate的那一套对象换成了对应的jpa对象。

**建议与Hibenrate的使用过程进行对比，地址**[Hibernate的原生xml方式开发和事务的使用](http://my.oschina.net/pingpangkuangmo/blog/404280#OSC_h2_12)

#3 jpa与spring的集成

上述jpa原生方式，还没有使用dataSource，为了引入dataSource来更好的管理数据库连接，为了简化jpa的配置，同时可以去掉jpa的核心配置文件，spring针对原生的jpa做了自己的集成工作。

##3.1 项目地址

-	[spring-springjpa-hibernate](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring/tree/master/spring-springjpa-hibernate)

##3.2 配置

-	第一步：配置数据库连接池dataSource

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

-	第二步：配置entityManagerFactory

		<bean id="entityManagerFactory"    	class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
			<property name="dataSource" ref="dataSource"/>
			<property name="jpaVendorAdapter" ref="jpaVendorAdapter"/>
			<property name="packagesToScan" value="com.demo.entity"/>
		</bean>
	
	其中的jpaVendorAdapter其实是收集配置参数，然后告诉LocalContainerEntityManagerFactoryBean所使用的PersistenceProvider，稍后详细分析，配置如下：
			
		<bean id="jpaVendorAdapter" class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
		    <property name="database" value="MYSQL"/>
		    <property name="showSql" value="true"/>
		    <property name="generateDdl" value="true"/>
		    <property name="databasePlatform" value="org.hibernate.dialect.MySQLDialect"/>
		</bean>

-	第三步：配置事务管理器JpaTransactionManager

		<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
			<property name="entityManagerFactory" ref="entityManagerFactory"></property>
		</bean>

	JpaTransactionManager是依赖于entityManagerFactory的

-	第四步：启动@Transactional注解的处理器

		<tx:annotation-driven proxy-target-class="true" transaction-manager="transactionManager"/>
	
	处理器依赖于transactionManager的

##3.3 使用案例

	@Repository
	public class UserDao {
	
		@PersistenceContext
		private EntityManager entityManager;
		
		@Transactional
		public void save(User user){
			entityManager.persist(user);
		}
	}

使用@PersistenceContext注解来获取entityManagerFactory创建的EntityManager对象，然后使用EntityManager进行增删改查

##3.4 原理分析

###3.4.1 如何创建EntityManagerFactory

我们可以看到spring是使用了一个工厂bean LocalContainerEntityManagerFactoryBean来创建entityManagerFactory。虽然配置的是一个工厂bean，但是容器在根据id来获取bean的时候，返回的是该工厂bean所创建的实体，即LocalContainerEntityManagerFactoryBean所创建的EntityManagerFactory。

spring创建EntityManagerFactory有2中方式，如下图所示：
![spring创建EntityManagerFactory有2中方式][1]

-	LocalEntityManagerFactoryBean	
	
	它分成2种情况来创建

	-	方式1：当LocalEntityManagerFactoryBean本身指定了PersistenceProvider，就使用该PersistenceProvider来创建EntityManagerFactory，详见上文的PersistenceProvider接口定义

	-	方式2：使用上文jpa原生方式的：
		
			EntityManagerFactory entityManagerFactory=Persistence.createEntityManagerFactory("test");
	
	LocalEntityManagerFactoryBean的源码如下：

		public class LocalEntityManagerFactoryBean extends AbstractEntityManagerFactoryBean {

			@Override
			protected EntityManagerFactory createNativeEntityManagerFactory() 
					throws PersistenceException {
				PersistenceProvider provider = getPersistenceProvider();
				if (provider != null) {
					// Create EntityManagerFactory directly through PersistenceProvider.
					EntityManagerFactory emf = provider.createEntityManagerFactory
						(getPersistenceUnitName(), getJpaPropertyMap());
					return emf;
				}
				else {
					// Let JPA perform its standard PersistenceProvider autodetection.
					return Persistence.createEntityManagerFactory(
						getPersistenceUnitName(), getJpaPropertyMap());
				}
			}
			//略了部分内容
		}

	我们再详细了解下下面的这个过程：

		EntityManagerFactory entityManagerFactory=Persistence.createEntityManagerFactory("test");

	它其实也是先获取所有的PersistenceProvider，然后遍历PersistenceProvider来创建EntityManagerFactory，源码如下：

		public static EntityManagerFactory createEntityManagerFactory(String 			
						persistenceUnitName, Map properties) {
			EntityManagerFactory emf = null;
			List<PersistenceProvider> providers = getProviders();
			for ( PersistenceProvider provider : providers ) {
				emf = provider.createEntityManagerFactory( persistenceUnitName, properties );
				if ( emf != null ) {
					break;
				}
			}
			if ( emf == null ) {
				throw new PersistenceException( "No Persistence provider 
					for EntityManager named " + persistenceUnitName );
			}
			return emf;
		}

	那它是如何来获取所有的PersistenceProvider的呢？

	这里就用到了<font color="red">**java的SPI机制(Service Provider Interfaces)**</font>。

	如下简单说明下，详细内容可以自行搜索：

	-	jpa定义了PersistenceProvider接口
	-	Hibernate要实现这个接口，在hibernate-entitymanager这个jar包中，在META-INF/services文件夹下，会有一个以PersistenceProvider接口全称命名的文件，如下图所示：

	![PersistenceProvider的SPI机制][2]

	文件里面的内容就是该接口对应的实现类，内容如下：

		org.hibernate.jpa.HibernatePersistenceProvider
		# The deprecated provider, logs warnings when used.
		org.hibernate.ejb.HibernatePersistence

	这就很容易方便jpa来寻找PersistenceProvider所有的实现类

-	LocalContainerEntityManagerFactoryBean

	它创建一个PersistenceProvider需要以下几个重要的属性元素
	
	-	jpaVendorAdapter

		它的主要作用就是收集一些配置信息，并且提供一个PersistenceProvider。接口定义如下：

			public interface JpaVendorAdapter {
				PersistenceProvider getPersistenceProvider();
				Map<String, ?> getJpaPropertyMap();
				//略
			}

		以HibernateJpaVendorAdapter为例，在它初始化的时候就会创建出HibernatePersistenceProvider，如下：
			
			public class HibernateJpaVendorAdapter extends AbstractJpaVendorAdapter
				private final PersistenceProvider persistenceProvider;
				public HibernateJpaVendorAdapter() {
					PersistenceProvider providerToUse;
					try {
						ClassLoader cl = HibernateJpaVendorAdapter.class.getClassLoader();
						Class<?> hibernatePersistenceProviderClass = cl.loadClass
							("org.hibernate.jpa.HibernatePersistenceProvider");
						providerToUse = (PersistenceProvider) hibernatePersistenceProviderClass.newInstance();
					}
					this.persistenceProvider = providerToUse;
					//略
				}
			}

	-	PersistenceProvider ：

		它的来历至少有两种方式：

		-	在配置LocalContainerEntityManagerFactoryBean的时候，直接配置一个PersistenceProvider
		-	如果没有进行上述配置，则使用上述的jpaVendorAdapter指定的PersistenceProvider

	-	dataSource
	
		不再像原生的jpa那样直接使用核心配置文件中的连接信息（这些连接信息是配置给PersistenceProvider的）

	-	packagesToScan等信息

		用于指定注解式实体所在的包路径，和dataSource一样都作为配置信息，最终都反应在PersistenceProvider创建的EntityManagerFactory上了

###3.4.2 如何获取EntityManager

我们看到在例子中，是通过使用@PersistenceContext来获取一个EntityManager的，我们知道这个EntityManager就是通过我们配置的上述EntityManagerFactory来创建的，但具体是一个什么过程呢？

突然发现这一块源码内容也好多，主要是PersistenceAnnotationBeanPostProcessor这个处理器在处理

@PersistenceContext注解和我们常用的@Autowired是类似的，他们都是实现依赖注入的，内容还是很多，所以打算之后另开一篇博客，单独讲解这类依赖注入的注解

至此我们就大致了解jpa与Spring是怎么集成的，总之还是通过PersistenceProvider和配置信息来创建出EntityManagerFactory。
	
	
###3.4.3 事务过程是怎么实现的

过程比较复杂，总之原理还是使用ThreadLocal机制

	@PersistenceContext
	private EntityManager entityManager;

	@Transactional
	public void save(User user){
		entityManager.persist(user);
		throw new RuntimeException();
	}

-	1 在执行save方法之前，使用了事务管理器JpaTransactionManager执行了开启事务的操作

	-	1.1 创建了一个EntityManager对象，以Hibernate来说就是创建了EntityManagerImpl（同时内部创建了Session和TransactionImpl），然后通过TransactionImpl开启事务
	-	1.2 将创建的EntityManagerImpl对象绑定到当前线程

-	2 通过注入进来的entityManager来保存user

	这个注入进来的entityManager仅仅是一个空壳子，是一个代理对象，它会获取当前线程绑定的上述EntityManagerImpl对象，来实现保存user

-	3 一旦出现异常，则使用1.1步骤中创建的EntityManagerImpl中的事务TransactionImpl来实现回滚

总之，保证了业务代码和事务代码使用的是同一个EntityManager对象对象，所以可以正常回滚。
总之使用@Transactional注解式的事务，总要使用ThreadLocal模式来保证业务代码和事务代码中的使用的connection是一致的这一原则。


#4 多数据源下jpa与spring集成开发和事务的使用

##4.1 项目地址

-	[spring-springjpa-hibernate-multDataSource](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring/tree/master/spring-springjpa-hibernate-multDataSource)

##4.2 配置

###4.2.1 配置两个数据库连接池dataSource

	<bean id="dataSource1"   
        class="com.mchange.v2.c3p0.ComboPooledDataSource"   
        destroy-method="close">   
        <property name="driverClass">   
            <value>${jdbc1.driverClass}</value>   
        </property>   
        <property name="jdbcUrl">   
            <value>${jdbc1.url}</value>   
        </property>   
        //略，见项目中的配置
    </bean>
    
     <!-- 配置数据源 -->
     <bean id="dataSource2"   
        class="com.mchange.v2.c3p0.ComboPooledDataSource"   
        destroy-method="close">   
        <property name="driverClass">   
            <value>${jdbc2.driverClass}</value>   
        </property>   
        <property name="jdbcUrl">   
            <value>${jdbc2.url}</value>   
        </property>   
        //略，见项目中的配置 
    </bean>

记得根据配置创建相应的数据库和表

###4.2.2 配置2个entityManagerFactory

	<bean id="jpaVendorAdapter" class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
	    <property name="database" value="MYSQL"/>
	    <property name="showSql" value="true"/>
	    <property name="generateDdl" value="true"/>
	    <property name="databasePlatform" value="org.hibernate.dialect.MySQLDialect"/>
	</bean>
    
    <bean id="entityManagerFactory1" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
		<property name="dataSource" ref="dataSource1"/>
		<property name="jpaVendorAdapter" ref="jpaVendorAdapter"/>
		<property name="packagesToScan" value="com.demo.entity"/>
	</bean>
	
	<bean id="entityManagerFactory2" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
		<property name="dataSource" ref="dataSource2"/>
		<property name="jpaVendorAdapter" ref="jpaVendorAdapter"/>
		<property name="packagesToScan" value="com.demo.entity"/>
		<property name="persistenceUnitName" value="test2"/>
	</bean>

这两个entityManagerFactory使用不同的dataSource，各自有一个persistenceUnitName名字(持久化单元的名字)，分别叫"test1"和"test2"。在上文中，并没有配置persistenceUnitName，采用默认值"default"

###4.2.3 配置2个事务管理器

	<bean id="transactionManager1" class="org.springframework.orm.jpa.JpaTransactionManager">
		<property name="entityManagerFactory" ref="entityManagerFactory1"></property>
	</bean>
	
	<bean id="transactionManager2" class="org.springframework.orm.jpa.JpaTransactionManager">
		<property name="entityManagerFactory" ref="entityManagerFactory2"></property>
	</bean>

###4.2.4 开启@Transactional的处理器

	<tx:annotation-driven proxy-target-class="true"/>

##4.3 使用方式

	@Repository
	public class UserDao {
	
		@PersistenceContext(unitName="test1")
		private EntityManager entityManager;
		@PersistenceContext(unitName="test2")
		private EntityManager entityManagerTest2;
		
		@Transactional("transactionManager1")
		public void save(User user){
			entityManager.persist(user);
		}
		@Transactional("transactionManager2")
		public void save2(User user){
			entityManagerTest2.persist(user);
		}
	}

@PersistenceContext(unitName="test1")表示使用persistenceUnitName="test1"的entityManagerFactory来创建EntityManager，同理@PersistenceContext(unitName="test2")也一样。

如果有多个entityManagerFactory，但是只使用@PersistenceContext没有指定unitName，则会报错，spring不知道该选择哪一个，所以需要指定unitName的名字

@Transactional("transactionManager1")表示使用id="transactionManager1"的JpaTransactionManager来作为事务管理器
同理@Transactional("transactionManager2")也一样。

多数据源就不再详细说明了

#5 spring-data-jpa的开发和事务的使用

##5.1 背景

引用IBM的一篇文章[使用 Spring Data JPA 简化 JPA 开发](http://www.ibm.com/developerworks/cn/opensource/os-cn-spring-jpa/?ca=drs-)

> Spring 对 JPA 的支持已经非常强大，开发者只需关心核心业务逻辑的实现代码，无需过多关注 EntityManager 的创建、事务处理等 JPA 相关的处理，这基本上也是作为一个开发框架而言所能做到的极限了。然而，Spring 开发小组并没有止步，他们再接再厉，于最近推出了 Spring Data JPA 框架，主要针对的就是 Spring 唯一没有简化到的业务逻辑代码，至此，开发者连仅剩的实现持久层业务逻辑的工作都省了，唯一要做的，就只是声明持久层的接口，其他都交给 Spring Data JPA 来帮你完成

> 至此，读者可能会存在一个疑问，框架怎么可能代替开发者实现业务逻辑呢？毕竟，每一个应用的持久层业务甚至领域对象都不尽相同，框架是怎么做到的呢？其实这背后的思想并不复杂，比如，当你看到 UserDao.findUserById() 这样一个方法声明，大致应该能判断出这是根据给定条件的 ID 查询出满足条件的 User 对象。Spring Data JPA 做的便是规范方法的名字，根据符合规范的名字来确定方法需要实现什么样的逻辑

这篇文章的内容已经很详细了，这里就仅仅详细地列出配置，真正跑起来

##5.2 配置

有了前面的基础后，就非常简单了

###5.2.1 配置dataSource

	<bean id="dataSource"   
        class="com.mchange.v2.c3p0.ComboPooledDataSource"   
        destroy-method="close">   
        <property name="driverClass">   
            <value>${jdbc.driverClass}</value>   
        </property>   
        <property name="jdbcUrl">   
            <value>${jdbc.url}</value>   
        </property>   
        //略   
    </bean>

###5.2.2 配置entityManagerFactory

	<bean id="jpaVendorAdapter" class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
	    <property name="database" value="MYSQL"/>
	    <property name="showSql" value="true"/>
	    <property name="generateDdl" value="true"/>
	    <property name="databasePlatform" value="org.hibernate.dialect.MySQLDialect"/>
	</bean>
    
    <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
		<property name="dataSource" ref="dataSource"/>
		<property name="jpaVendorAdapter" ref="jpaVendorAdapter"/>
		<property name="packagesToScan" value="com.demo.entity"/>
	</bean>

###5.2.3 配置transactionManager

	<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
		<property name="entityManagerFactory" ref="entityManagerFactory"></property>
	</bean>

###5.2.4 开启@Transactional的处理器

	<tx:annotation-driven proxy-target-class="true" transaction-manager="transactionManager"/>

###5.2.5 配置要扫描的dao的路径

	<jpa:repositories base-package="com.demo.dao"/>

使用jpa命名空间的元素，需要加入如下jpa的约束配置：

	<?xml version="1.0" encoding="UTF-8" ?>
	<beans xmlns="http://www.springframework.org/schema/beans" 
		xmlns:jpa="http://www.springframework.org/schema/data/jpa"
		//略
		xsi:schemaLocation="http://www.springframework.org/schema/beans
	    http://www.springframework.org/schema/data/jpa
	    http://www.springframework.org/schema/data/jpa/spring-jpa-1.2.xsd">

我们之前需要自己手写一个UserDao，自己实现相应的方法，注入EntityManager，然后进行增删改查，现在不需要了，只需定义一个接口即可

	public interface UserDao extends CrudRepository<User,Long>{

	}

CrudRepository<User,Long> 类型中的前者User表示User实体，后者Long表示User实体的主键类型

##5.3 使用过程

我们只需定义上述一个接口，即可在别的地方注入使用UserDao，来进行增删改查，如下:

	@Autowired
	private UserDao userDao;
	
	@Test
	public void testSaveUser(){
		User user=new User();
		user.setName("王五");
		user.setAge(22);
		userDao.save(user);
	}

##5.4 spring-data-jpa实现的功能介绍

上面仅仅是一个简单的使用例子，更多复杂的例子，见IBM的一篇文章[使用 Spring Data JPA 简化 JPA 开发](http://www.ibm.com/developerworks/cn/opensource/os-cn-spring-jpa/?ca=drs-)

[1]: http://static.oschina.net/uploads/space/2015/0421/073016_K8Bs_2287728.png
[2]: http://static.oschina.net/uploads/space/2015/0421/193110_JzAd_2287728.png