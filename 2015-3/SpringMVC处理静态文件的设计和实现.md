#1 静态文件的处理前言分析

最近想把SpringMVC对于静态资源的处理策略弄清楚，如它和普通的请求有什么区别吗？

有人可能就要说了，现在有些静态资源都不是交给这些框架来处理，而是直接交给容器来处理，这样更加高效。我想说的是，即使是这样，处理静态资源也是MVC框架应该提供的功能，而不是依靠外界。

这里以tomcat容器中的SpringMVC项目为例。整个静态资源的访问，效果图如下：

![资源访问图][1]

可以分成如下2个大的过程

-	tomcat根据url-pattern选择servlet的过程
-	SpringMVC对静态资源的处理过程（这个留到下一篇文章来详细的源码说明）


#tomcat的处理策略
这里要看tomcat的源码，所以pom中加入相应依赖，使debug的时候能够定位到源码文件，目前我所使用的tomcat版本为7.0.55，你要是使用的不同版本，则更换下对应依赖的版本就行

	<dependency>
		<groupId>org.apache.tomcat</groupId>
		<artifactId>tomcat-catalina</artifactId>
		<version>7.0.55</version>
		<scope>provided</scope>
	</dependency>
	<dependency>
		<groupId>org.apache.tomcat</groupId>
		<artifactId>tomcat-coyote</artifactId>
		<version>7.0.55</version>
		<scope>provided</scope>
	</dependency>
	<dependency>
	    <groupId>org.apache.tomcat</groupId>
	    <artifactId>tomcat-jasper</artifactId>
	    <version>7.0.55</version>
		<scope>provided</scope>
	</dependency>

##tomcat默认注册的servlet
tomcat默认注册了，映射 '/' 路径的的DefaultServlet，映射*.jsp和*.jspx的JspServlet，这些内容配置在tomcat的conf/web.xml文件中，如下：

	<servlet>
        <servlet-name>default</servlet-name>
        <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    </servlet>

	<servlet>
        <servlet-name>jsp</servlet-name>
        <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
    </servlet>

	<servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <servlet-mapping>
        <servlet-name>jsp</servlet-name>
        <url-pattern>*.jsp</url-pattern>
        <url-pattern>*.jspx</url-pattern>
    </servlet-mapping>

-	DefaultServlet可以用来处理tomcat一些资源文件
-	JspServlet则用来处理一些jsp文件,对这些jsp文件进行一些翻译

我们可以修改此配置文件，来添加或者删除一些默认的servlet配置，举个简单例子：tomcat的根路径下有一个a.jsp文件，这里再说明下，如果使用的是eclipse开发，则我们在其中新建的tomcat server的一些信息如下（这点很重要，有时候修改tomcat配置没起作用就是因为你修改的地方不对导致的）：

-	新建的tomcat server，是将你所安装的tomcat的配置进行复制后，存放在当前eclipse所在工作空间路径的server项目下，如下：
![新建tomcat路径][2] 

所以以后要修改所使用的tomcat信息，就直接在该项目下修改，或者直接去该项目的路径下，直接修改对应的配置文件

-	新建的tomcat server的运行环境不是你所安装的tomcat的webapps目录下，而是在当前eclipse所在的工作空间的.metadata文件下，具体如下：   .metadata\\.plugins\org.eclipse.wst.server.core ，这个目录下会有一个或多个tmp目录，每个tmp目录都对应着一个tomcat的真实运行环境，然后找到那个你所使用的tmp目录，你就会看到如下的信息
![tomcat临时运行环境][3]

这里的wtwebapps就是tomcat发布的根目录。

在这个根目录中，我们放一个jsp文件，文件内容如下：

	<%@page contentType="text/html"%>
	<%@page pageEncoding="UTF-8"%>
	<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
	   "http://www.w3.org/TR/html4/loose.dtd">
	<html>
	    <head>
	        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
	        <title>JSP Page</title>
	    </head>
	    <body>
	    <h1>JSP Page</h1>
	        Hello ${param.name}!
	    </body>
	</html>

默认情况下，即JspServlet存在，访问 http://localhost:8080/a.jsp?name=lg ，结果如下：
![jsp作为jsp的访问结果][4]

如果你修改了tomcat的默认配置，去掉JspServlet的话，同样访问 http://localhost:8080/a.jsp?name=lg ，结果如下
![jsp作为一般资源文件的访问结果][5]

这时候就，没有了JspServlet，不会进行相应的翻译工作，而是使用DefaultServlet直接将该文件内容进行返回。

在有JspServlet的情况下，JspServlet优先处理jsp文件，当不存在的情况下，才使用DefaultServlet来处理，他们到底是按照什么样的执行顺序呢？这就需要来说说在web.xml中配置的url-pattern了

##servlet的url-pattern的规则
对于servlet的url-pattern规则，这里也有一篇对应的源码分析文章[tomcat的url-pattern源码分析](http://www.cnblogs.com/fangjian0423/p/servletContainer-tomcat-urlPattern.html)。

###tomcat源码中的几个概念
在分析之前简单看下tomcat源码中的几个概念，Context、Wrapper、Servlet：

-	Servlet 这个很清楚，就是继承了HttpServlet，用户用它的service方法来处理请求

-	Wrapper 则是Servlet和映射的结合，具体点就是web.xml中配置的servlet信息

		<servlet>
			<servlet-name>mvc</servlet-name>
			<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
			<load-on-startup>1</load-on-startup>
		</servlet>
	
		<servlet-mapping>
			<servlet-name>mvc</servlet-name>
			<url-pattern>/*</url-pattern>
		</servlet-mapping>


-	Context 表示一个应用，包含了web.xml中配置的所有信息



##案例分析

##filter的url-pattern的规则




  [1]: http://static.oschina.net/uploads/space/2015/0307/162637_vRZz_2287728.png
  [2]: http://static.oschina.net/uploads/space/2015/0307/174141_BGq7_2287728.png
  [3]: http://static.oschina.net/uploads/space/2015/0307/175150_s9g8_2287728.png
  [4]: http://static.oschina.net/uploads/space/2015/0307/180220_vYEc_2287728.png
  [5]: http://static.oschina.net/uploads/space/2015/0307/180600_Lpf7_2287728.png