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

长长的匹配规则，有兴趣的可以去仔细研究下，对于Welcome resources匹配，下文会举2个例子来详细的分析其规则，其他的我们仅仅了解下大概的匹配顺序就可以了，匹配顺序如下：

-	(1) 首先精准匹配

-	(2) 然后是通配符匹配

-	(3) 然后是扩展名匹配

-	(4) 然后是欢迎页面匹配（这里又细分了很多的规则，下面的案例分析会详细说明）

-	(5) 最后是默认匹配


#3 案例分析（结合源码）

在说明案例之前，需要先将eclipse中的tomcat信息说明白，有时候修改tomcat配置没起作用就是因为你修改的地方不对导致的

##3.1 前提：eclipse中tomcat的配置信息
-	新建的tomcat server，是将你所安装的tomcat的配置进行复制后，存放在当前eclipse所在工作空间路径的server项目下，如下：
![新建tomcat路径][2] 

所以以后要修改所使用的tomcat信息，就直接在该项目下修改，或者直接去该项目的路径下，直接修改对应的配置文件

-	新建的tomcat server的运行环境不是你所安装的tomcat的webapps目录下，而是在当前eclipse所在的工作空间的.metadata文件下，具体如下：   .metadata\\.plugins\org.eclipse.wst.server.core ，这个目录下会有一个或多个tmp目录，每个tmp目录都对应着一个tomcat的真实运行环境，然后找到那个你所使用的tmp目录，你就会看到如下的信息
![tomcat临时运行环境][3]

这里的wtwebapps就是tomcat默认的发布根目录，这个是不固定的，可配置的。

##3.2 jsp的访问案例

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

因为tomcat默认配置了，映射 / 的DefaultServlet和映射 *.jsp 的JspServlet。在初始化web.xml的时候，上文讲的Mapper类按照归类规则，DefaultServlet作为了默认的servlet，JspServlet作为了扩展名的servlet，它比DefaultServlet的级别高，执行了扩展名匹配。当去掉JspServlet时，使用了DefaultServlet，执行了默认匹配，此时的jsp文件仅仅是一个一般的资源文件。

##3.3 welcome-file-list案例

它是具有两种作用的，作为项目的主页和作为跳转的阶梯，下面先介绍两个案例，然后根据源码分析其原因。

注意点： 我把项目的根目录作为tomcat的发布的目录，所以访问 http://localhost:8080/ 中不再加入项目名

###3.3.1 作为项目的主页

-	案例1：在项目的根路径下，放置一个a.html文件，FirstServlet拦截 /first/*，SecondServlet拦截 *.action，web.xml中是如下配置

		<servlet>
			<servlet-name>first</servlet-name>
			<servlet-class>com.lg.servlet.FirstServlet</servlet-class>
			<load-on-startup>1</load-on-startup>
		</servlet>
	
		<servlet-mapping>
			<servlet-name>first</servlet-name>
			<url-pattern>/first/*</url-pattern>
		</servlet-mapping>
		
		<servlet>
			<servlet-name>second</servlet-name>
			<servlet-class>com.lg.servlet.SecondServlet</servlet-class>
			<load-on-startup>1</load-on-startup>
		</servlet>
	
		<servlet-mapping>
			<servlet-name>second</servlet-name>
			<url-pattern>*.action</url-pattern>
		</servlet-mapping>	
		<welcome-file-list>
			<welcome-file>a.html</welcome-file>
		</welcome-file-list>

	这时，我们访问 http://localhost:8080/ 即想访问项目的主页，就能访问到a.html的内容，如下：
	![访问项目主页a.html][6]

-	案例2：对welcome-file-list稍加修改，其他不变,如下

		<welcome-file-list>
			<welcome-file>a.jsp</welcome-file>
		</welcome-file-list>

	在根目录下再存放一个a.jsp文件，如下：
	
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
	
	访问 http://localhost:8080/ 结果返回的是

	![访问项目主页a.jsp][7]

	可以得出表面结论： 就是tomcat会根据不同的扩展名，使用相应的servlet来解析文件，然后返回

###3.3.2 作为跳转的阶梯

-	案例3：对welcome-file-list再次修改如下，其他不变：

		<welcome-file-list>
			<welcome-file>a.action</welcome-file>
		</welcome-file-list>
	
	这里又可以分2种情况，即根目录下是否存在a.action，然而无论哪种情况，访问http://localhost:8080/，在本案例中都会返回上文配置的SecondServlet的内容（然而他们的执行逻辑却是不一样的），返回内容如下：
	
	![访问项目主页跳转到SecondServlet][8]
	
	
-	案例4：同理，再次修改welcome-file-list如下，访问http://localhost:8080/，就可以访问到上文配置的FirstServlet的内容：
	
		<welcome-file-list>
			<welcome-file>first/abc</welcome-file>
		</welcome-file-list>

###3.3.3 源码解释

下面我们就来根据源码分析分析整个是什么样的过程，这一部分详细的源码如下：

	
	// Rule 4 -- Welcome resources processing for servlets
    if (mappingData.wrapper == null) {
        boolean checkWelcomeFiles = checkJspWelcomeFiles;
        if (!checkWelcomeFiles) {
            char[] buf = path.getBuffer();
            checkWelcomeFiles = (buf[pathEnd - 1] == '/');
        }
        if (checkWelcomeFiles) {
            for (int i = 0; (i < contextVersion.welcomeResources.length)
                     && (mappingData.wrapper == null); i++) {
                path.setOffset(pathOffset);
                path.setEnd(pathEnd);
                path.append(contextVersion.welcomeResources[i], 0,
                        contextVersion.welcomeResources[i].length());
                path.setOffset(servletPath);

                // Rule 4a -- Welcome resources processing for exact macth
                internalMapExactWrapper(exactWrappers, path, mappingData);

                // Rule 4b -- Welcome resources processing for prefix match
                if (mappingData.wrapper == null) {
                    internalMapWildcardWrapper
                        (wildcardWrappers, contextVersion.nesting,
                         path, mappingData);
                }

                // Rule 4c -- Welcome resources processing
                //            for physical folder
                if (mappingData.wrapper == null
                    && contextVersion.resources != null) {
                    Object file = null;
                    String pathStr = path.toString();
                    try {
                        file = contextVersion.resources.lookup(pathStr);
                    } catch(NamingException nex) {
                        // Swallow not found, since this is normal
                    }
                    if (file != null && !(file instanceof DirContext) ) {
                        internalMapExtensionWrapper(extensionWrappers, path,
                                                    mappingData, true);
                        if (mappingData.wrapper == null
                            && contextVersion.defaultWrapper != null) {
                            mappingData.wrapper =
                                contextVersion.defaultWrapper.object;
                            mappingData.requestPath.setChars
                                (path.getBuffer(), path.getStart(),
                                 path.getLength());
                            mappingData.wrapperPath.setChars
                                (path.getBuffer(), path.getStart(),
                                 path.getLength());
                            mappingData.requestPath.setString(pathStr);
                            mappingData.wrapperPath.setString(pathStr);
                        }
                    }
                }
            }

            path.setOffset(servletPath);
            path.setEnd(pathEnd);
        }

    }
	// Rule 4d --我暂且叫它 Rule 4d (源码并没有这样写)
    if (mappingData.wrapper == null) {
        boolean checkWelcomeFiles = checkJspWelcomeFiles;
        if (!checkWelcomeFiles) {
            char[] buf = path.getBuffer();
            checkWelcomeFiles = (buf[pathEnd - 1] == '/');
        }
        if (checkWelcomeFiles) {
            for (int i = 0; (i < contextVersion.welcomeResources.length)
                     && (mappingData.wrapper == null); i++) {
                path.setOffset(pathOffset);
                path.setEnd(pathEnd);
                path.append(contextVersion.welcomeResources[i], 0,
                            contextVersion.welcomeResources[i].length());
                path.setOffset(servletPath);
                internalMapExtensionWrapper(extensionWrappers, path,
                                            mappingData, false);
            }

            path.setOffset(servletPath);
            path.setEnd(pathEnd);
        }
    }

从上面可以看到，其中的path不再是原来的path,而是我们的访问path+welcome-file中配置的路径，作为全新的路径，对于欢迎资源，又细分了4中规则，分别如下：

-	4a: 对全新的路径进行精准匹配
-	4b: 对全新的路径进行通配符匹配
-	4c: 根据全新的路径，进行查找是否存在相应的文件，如果存在相应的文件，则需要将该文件返回。在返回前我们需要进一步确认，这个文件是不是讲文件内容源码返回，还是像jsp文件一样，进行一定的处理然后再返回，所以又要确认下文件的扩展名是怎样的

	-	4c1: 尝试寻找能够处理该文件扩展名的servlet，即进行扩展名匹配,如果找到，则使用对应的servlet
	-	4c2: 如果没找到，则默认使用defaultWrapper，即DefaultServlet（它只会将文件内容源码返回，不做任何处理）
	
-	4d: 对全新的路径进行扩展名匹配（与4c的目的不同，4c的主要目的是想返回一个文件的内容，在返回内容前涉及到扩展名匹配）

有了以上的规则，我们就来详细看看上文的4个案例都是走的哪个规则

-	案例1： a.html，4a、4b没有匹配到，到4c的时候，找到了该文件，然后又尝试扩展名匹配，来决定是走4c1还是4c2，由于.html还没有对应的servlet来处理，就使用了默认的DefaultServlet
-	案例2： a.jsp，同上，在走到4c的时候，找到了处理.jsp对应的servlet，所以走了4c1
-	案例3： a.action,如果根目录下有a.action文件，则走到4c1的时候，进行扩展名匹配，匹配到了SecondServlet，即走了4c1，使用SecondServlet来处理请求；如果根目录下没有a.action文件，则走到了4d，进行扩展名匹配，同样匹配到了SecondServlet，即走了4d，同样使用SecondServlet来处理请求
-	案例4： first/abc，执行4b的时候，就匹配到了FirstServlet，所以使用FirstServlet来处理请求

至此，就把welcome-file-list彻底讲清楚了，有什么问题和疑问，欢迎提问

#3 结束语

了解了tomcat的url-pattern的规则后，下一篇文章就要说明SpringMVC是如何来处理静态资源的，以及他们的综合分析。

  [1]: http://static.oschina.net/uploads/space/2015/0307/162637_vRZz_2287728.png
  [2]: http://static.oschina.net/uploads/space/2015/0307/174141_BGq7_2287728.png
  [3]: http://static.oschina.net/uploads/space/2015/0307/175150_s9g8_2287728.png
  [4]: http://static.oschina.net/uploads/space/2015/0307/180220_vYEc_2287728.png
  [5]: http://static.oschina.net/uploads/space/2015/0307/180600_Lpf7_2287728.png
  [6]: http://static.oschina.net/uploads/space/2015/0309/220310_gGkb_2287728.png
  [7]: http://static.oschina.net/uploads/space/2015/0309/220329_3dMF_2287728.png
  [8]: http://static.oschina.net/uploads/space/2015/0309/221414_1njo_2287728.png