#1 静态文件的处理前言分析

最近想把SpringMVC对于静态资源的处理策略弄清楚，如它和普通的请求有什么区别吗？

有人可能就要说了，现在有些静态资源都不是交给这些框架来处理，而是直接交给容器来处理，这样更加高效。我想说的是，即使是这样，处理静态资源也是MVC框架应该提供的功能，而不是依靠外界。

这里以tomcat容器中的SpringMVC项目为例。整个静态资源的访问，效果图如下：

![资源访问图][1]

可以分成如下2个大的过程

-	tomcat根据url-pattern选择servlet的过程
-	SpringMVC对静态资源的处理过程（这个留到下一篇文章来详细的源码说明）

#2 tomcat的处理策略
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

##2.1 tomcat默认注册的servlet
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

我们可以修改此配置文件，来添加或者删除一些默认的servlet配置。

下面来看下这些servlet的url-pattern的规则是什么样的

##2.2 servlet的url-pattern的规则
对于servlet的url-pattern规则，这里也有一篇对应的源码分析文章[tomcat的url-pattern源码分析](http://www.cnblogs.com/fangjian0423/p/servletContainer-tomcat-urlPattern.html)。

###2.2.1 tomcat源码中的几个概念
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


-	Context 表示一个应用，包含了web.xml中配置的所有信息，所以当一个请求到来时，它负责找到对应的Servlet，然后调用这个Servlet的service方法，执行我们所写的业务逻辑。

Context把上述的根据映射寻找Servlet的过程封装起来交给了一个org.apache.tomcat.util.http.mapper.Mapper类来完成，所以请求匹配规则都在这个Mapper中来完成。

所以这个Mapper做了2件事情

-	在初始化web.xml的时候，Mapper需要收集其中的servlet及其映射信息并进行一定的处理，存储到Mapper的内部类ContextVersion中

-	在请求到来的时候，它能根据请求地址，选择出对应的servlet等信息，供使用

Mapper的内部类ContextVersion对映射对应的servlet进行了分类存储，如下：

	protected static final class ContextVersion extends MapElement {
        public String[] welcomeResources = new String[0];
        public Wrapper defaultWrapper = null;
        public Wrapper[] exactWrappers = new Wrapper[0];
        public Wrapper[] wildcardWrappers = new Wrapper[0];
        public Wrapper[] extensionWrappers = new Wrapper[0];
       	//略
	}
总共分成了5种，分别是

-	welcomeResources 欢迎页面，就是web.xml中可以配置的如下内容，待会以案例的形式详细说明它的作用

		<welcome-file-list>
			<welcome-file>index.html</welcome-file>
			<welcome-file>index.htm</welcome-file>
		</welcome-file-list>

-	defaultWrapper 用于存放默认的servlet信息

-	exactWrappers 用于精确匹配，即要求必须一模一样

-	wildcardWrappers 用于通配符匹配 如 /*、/abc/\*

-	extensionWrappers 用于扩展名匹配，即 *.jsp、*.html等	

下面就来看看Mapper是如何进行归类处理的

###2.2.2 Mapper的归类处理Servlet和映射信息

	protected void addWrapper(ContextVersion context, String path,
            Object wrapper, boolean jspWildCard, boolean resourceOnly) {

        synchronized (context) {
            if (path.endsWith("/*")) {
                // Wildcard wrapper
                String name = path.substring(0, path.length() - 2);
                Wrapper newWrapper = new Wrapper(name, wrapper, jspWildCard,
                        resourceOnly);
                Wrapper[] oldWrappers = context.wildcardWrappers;
                Wrapper[] newWrappers =
                    new Wrapper[oldWrappers.length + 1];
                if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                    context.wildcardWrappers = newWrappers;
                    int slashCount = slashCount(newWrapper.name);
                    if (slashCount > context.nesting) {
                        context.nesting = slashCount;
                    }
                }
            } else if (path.startsWith("*.")) {
                // Extension wrapper
                String name = path.substring(2);
                Wrapper newWrapper = new Wrapper(name, wrapper, jspWildCard,
                        resourceOnly);
                Wrapper[] oldWrappers = context.extensionWrappers;
                Wrapper[] newWrappers =
                    new Wrapper[oldWrappers.length + 1];
                if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                    context.extensionWrappers = newWrappers;
                }
            } else if (path.equals("/")) {
                // Default wrapper
                Wrapper newWrapper = new Wrapper("", wrapper, jspWildCard,
                        resourceOnly);
                context.defaultWrapper = newWrapper;
            } else {
                // Exact wrapper
                final String name;
                if (path.length() == 0) {
                    // Special case for the Context Root mapping which is
                    // treated as an exact match
                    name = "/";
                } else {
                    name = path;
                }
                Wrapper newWrapper = new Wrapper(name, wrapper, jspWildCard,
                        resourceOnly);
                Wrapper[] oldWrappers = context.exactWrappers;
                Wrapper[] newWrappers =
                    new Wrapper[oldWrappers.length + 1];
                if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                    context.exactWrappers = newWrappers;
                }
            }
        }
    }

上面几个if else语句就解释的很清楚

-	以 /* 结尾的，都纳入通配符匹配，存到ContextVersion的wildcardWrappers中

-	以 *.开始的，都纳入扩展名匹配中，存到ContextVersion的extensionWrappers中

-	/  ，作为默认的，存到ContextVersion的defaultWrapper中

-	其他的都作为精准匹配，存到ContextVersion的exactWrappers中

此时我们可能会想，url形式多样，也不会仅仅只有这几种吧。如/a/\*.jsp，即不是以 /* 结尾，也不是以 \*. 开始，貌似只能分配到精准匹配中去了，这又不太合理吧。实际上tomcat就把url形式限制死了，它会进行相应的检查，如下

	private boolean validateURLPattern(String urlPattern) {

        if (urlPattern == null)
            return (false);
        if (urlPattern.indexOf('\n') >= 0 || urlPattern.indexOf('\r') >= 0) {
            return (false);
        }
        if (urlPattern.equals("")) {
            return true;
        }
        if (urlPattern.startsWith("*.")) {
            if (urlPattern.indexOf('/') < 0) {
                checkUnusualURLPattern(urlPattern);
                return (true);
            } else
                return (false);
        }
        if ( (urlPattern.startsWith("/")) &&
                (urlPattern.indexOf("*.") < 0)) {
            checkUnusualURLPattern(urlPattern);
            return (true);
        } else
            return (false);

    }

显然，urlPattern可以为"",其他必须以 *. 或者 / 开头，并且两者不能同时存在。/a/\*.jsp不符合最后一个条件，直接报错，tomcat启动失败，所以我们不用过多的担心servlet标签中的url-pattern的复杂性。

初始化归类完成之后，当请求到来时，就需要利用已归类好的数据进行匹配了，找到合适的Servlet来响应

###2.2.3 Mapper匹配请求对应的Servlet

在Mapper的internalMapWrapper方法中，存在着匹配规则，如下

	private final void internalMapWrapper(ContextVersion contextVersion,
                                          CharChunk path,
                                          MappingData mappingData)
        throws Exception {
		//略
        // Rule 1 -- Exact Match
        Wrapper[] exactWrappers = contextVersion.exactWrappers;
        internalMapExactWrapper(exactWrappers, path, mappingData);

        // Rule 2 -- Prefix Match
        boolean checkJspWelcomeFiles = false;
        Wrapper[] wildcardWrappers = contextVersion.wildcardWrappers;
        if (mappingData.wrapper == null) {
            internalMapWildcardWrapper(wildcardWrappers, contextVersion.nesting,
                                       path, mappingData);
            //略
        }
		//略
        // Rule 3 -- Extension Match
        Wrapper[] extensionWrappers = contextVersion.extensionWrappers;
        if (mappingData.wrapper == null && !checkJspWelcomeFiles) {
            internalMapExtensionWrapper(extensionWrappers, path, mappingData,
                    true);
        }

        // Rule 4 -- Welcome resources processing for servlets
        if (mappingData.wrapper == null) {
            boolean checkWelcomeFiles = checkJspWelcomeFiles;
            //略
        }
        //略
        // Rule 7 -- Default servlet
        if (mappingData.wrapper == null && !checkJspWelcomeFiles) {
            if (contextVersion.defaultWrapper != null) {
                mappingData.wrapper = contextVersion.defaultWrapper.object;
                mappingData.requestPath.setChars
                    (path.getBuffer(), path.getStart(), path.getLength());
                mappingData.wrapperPath.setChars
                    (path.getBuffer(), path.getStart(), path.getLength());
            }
           //略
        }
		//略
    }

长长的匹配规则，有兴趣的可以去仔细研究下，我们仅仅了解下大概的匹配顺序就可以了，匹配顺序如下：

-	(1) 首先精准匹配

-	(2) 然后是通配符匹配

-	(3) 然后是扩展名匹配

-	(4) 然后是欢迎页面匹配

-	(5) 最后是默认匹配


#3 案例分析（结合源码）

在说明案例之前，需要先将eclipse中的tomcat信息说明白，有时候修改tomcat配置没起作用就是因为你修改的地方不对导致的

##3.1 eclipse中tomcat的配置信息
-	新建的tomcat server，是将你所安装的tomcat的配置进行复制后，存放在当前eclipse所在工作空间路径的server项目下，如下：
![新建tomcat路径][2] 

所以以后要修改所使用的tomcat信息，就直接在该项目下修改，或者直接去该项目的路径下，直接修改对应的配置文件

-	新建的tomcat server的运行环境不是你所安装的tomcat的webapps目录下，而是在当前eclipse所在的工作空间的.metadata文件下，具体如下：   .metadata\\.plugins\org.eclipse.wst.server.core ，这个目录下会有一个或多个tmp目录，每个tmp目录都对应着一个tomcat的真实运行环境，然后找到那个你所使用的tmp目录，你就会看到如下的信息
![tomcat临时运行环境][3]

这里的wtwebapps就是tomcat默认的发布根目录，这个是不固定的，可配置的。

##3.1 jsp的访问案例

举个简单例子：tomcat的根路径下有一个a.jsp文件，就是上述的tomcat发布的根目录，在这个根目录中，我们放一个jsp文件，文件内容如下：

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

因为tomcat默认配置了，映射 / 的DefaultServlet和映射 *.jsp 的JspServlet。在初始化web.xml的时候，上文讲的Mapper类按照归类规则，DefaultServlet作为了默认的servlet，JspServlet作为了扩展名的servlet，比DefaultServlet的级别高，执行了扩展名匹配。当去掉JspServlet时，执行了默认匹配，此时的jsp文件仅仅是一个一般的资源文件。

##3.2 welcome-file-list的作用




  [1]: http://static.oschina.net/uploads/space/2015/0307/162637_vRZz_2287728.png
  [2]: http://static.oschina.net/uploads/space/2015/0307/174141_BGq7_2287728.png
  [3]: http://static.oschina.net/uploads/space/2015/0307/175150_s9g8_2287728.png
  [4]: http://static.oschina.net/uploads/space/2015/0307/180220_vYEc_2287728.png
  [5]: http://static.oschina.net/uploads/space/2015/0307/180600_Lpf7_2287728.png