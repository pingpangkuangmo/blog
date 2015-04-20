#1 前面的文章索引

-	[jdbc的使用和事务]()
-	[JdbcTemplate的使用和事务]()
-	[Hibernate的原生xml方式开发和事务的使用]()
-	[Hibernate与spring集成的xml方式开发和事务的使用]()
-	[Hibernate的原生注解方式开发和事务的使用]()
-	[Hibernate与spring集成的注解方式开发和事务的使用]()

#2 jpa原生开发

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

-	PersistenceProvider接口： 

	根据核心配置文件能创建出EntityManagerFactory

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

	jpa定义的事务接口，其实可以类似于Hibernate的Transaction接口。对于Hibernate来说，其实它就是对Transaction的封装。Hibernate实现的EntityTransaction是TransactionImpl，源码如下：

		public class TransactionImpl implements EntityTransaction {

			private Transaction tx;
			//略
		}

##2.5 jpa的使用案例

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

我们可以看到，和Hibernate的过程非常相似，只不过把Hibernate的那一套对象换成了对应的jpa对象。

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

	我们可以看到spring是使用了一个工厂bean来创建entityManagerFactory。虽然配置的是一个工厂bean，但是容器在根据id来获取bean的时候，返回的是该工厂bean所创建的实体，即LocalContainerEntityManagerFactoryBean所创建的EntityManagerFactory

	


