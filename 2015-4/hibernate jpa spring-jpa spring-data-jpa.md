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

-	spring-hibernate-xml： hibernate的原生xml方式开发
	
-	spring-hibernate-xml-tx：hibernate的原生xml方式的事务的使用
	
-	spring-hibernate-annotation：hibernate的原生的注解方式开发
	
-	spring-hibernate-annotation-tx：hibernate的原生的注解方式的事务的使用
	
-	spring-jpa-hibernate：jpa的原生xml方式开发和事务的使用

-	spring-springjpa-hibernate:jpa的spring集成方式开发和事务的使用

-	spring-data-jpa-hibernate：spring-data-jpa方式开发和事务的使用

案例都是使用SpringMVC搭建的环境，主要是可以在此基础上方便开发。而学习过程的测试案例都是使用的是spring-test来进行测试的。

项目地址如下[hibernate-jpa-spring](https://git.oschina.net/pingpangkuangmo/hibernate-jpa-spring#hibernate-jpa-spring-三者的千丝万缕的关系)

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
##3.1 hibernate的原生xml方式开发

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