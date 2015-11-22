#1 参考如下文章整理的笔记

-	[Cas简介](http://www.iteye.com/blogs/subjects/cas168)

#2 CAS实现SSO的原理图

![CAS实现SSO的原理图](https://static.oschina.net/uploads/img/201511/04073455_iD5D.png "CAS实现SSO的原理图")

#3 AuthenticationEntryPoint用户登录入口

ExceptionTranslationFilter拥有该对象，在捕获到AuthenticationException和AccessDeniedException异常时，如果是未登录状态则使用AuthenticationEntryPoint引导用户到登陆界面，接口定义如下：

	public interface AuthenticationEntryPoint {
    
		//准备认证过程
	    void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException)
	        throws IOException, ServletException;
	}

实现情况如下：

![AuthenticationEntryPoint接口实现](https://static.oschina.net/uploads/img/201511/04074250_NAX4.png "AuthenticationEntryPoint接口实现")

默认实现LoginUrlAuthenticationEntryPoint，就是指定一个重定向地址，改地址不能被拦截，如下配置

	<security:http auto-config="true">
      <security:form-login login-page="/login.html"
         login-processing-url="/login.do" username-parameter="username"
         password-parameter="password" />
      <!-- 表示匿名用户可以访问 -->
      <security:intercept-url pattern="/login.html" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
      <security:intercept-url pattern="/**" access="ROLE_USER" />
    </security:http>

如果是CAS实现即CasAuthenticationEntryPoint：

也很简单就是重定向到CAS Server所在地址，同时带上目前要访问的serviceUrl。

总体地址就是：casServerLoginUrl？serviceParameterName=URLEncoder.encode(serviceUrl, "UTF-8")；

上述serviceParameterName来源是ServiceProperties对象，该对象用户保存和CAS Server交流的属性名称，默认如下：

	public class ServiceProperties implements InitializingBean {

	    public static final String DEFAULT_CAS_ARTIFACT_PARAMETER = "ticket";
	
	    public static final String DEFAULT_CAS_SERVICE_PARAMETER = "service";
	}

CasAuthenticationEntryPoint拥有上述ServiceProperties对象：

	public class CasAuthenticationEntryPoint implements AuthenticationEntryPoint, InitializingBean {
   
	    private ServiceProperties serviceProperties;
	
		//即CAS Server的登陆地址
	    private String loginUrl;
	}

#4 CasAuthenticationProvider对用户的ticket请求进行验证

认证过程如下：

	final Assertion assertion = this.ticketValidator.validate(authentication.getCredentials().toString(), getServiceUrl(authentication));
    final UserDetails userDetails = loadUserByAssertion(assertion);
    userDetailsChecker.check(userDetails);
    return new CasAuthenticationToken(this.key, userDetails, authentication.getCredentials(),
                    authoritiesMapper.mapAuthorities(userDetails.getAuthorities()), userDetails, assertion);

-	1 使用内部对象TicketValidator ticketValidator向CAS Server发送验证请求，参数是ticket和serviceUrl。CAS Server会返回xml内容作为响应，处理响应构建一个Assertion对象

-	2 再对上述Assertion对象进行检查是否被锁等，最后封装成一个CasAuthenticationToken对象，CasAuthenticationProvider认证结束

之后要做的就是把上述认证结果保存到session中。谁来做？应该是之前的Filter SecurityContextPersistenceFilter,它专门负责将认证信息存入session中。

#CasAuthenticationFilter 掌控执行认证过程

![认证Filter](https://static.oschina.net/uploads/img/201511/04083047_lIbI.png "认证Filter")

我们已经知道UsernamePasswordAuthenticationFilter、CasAuthenticationFilter两者都有一个是否需要认证的判断逻辑

前者就是判断是否是要处理的登陆地址，后者判断请求是否含有ticket参数

然后就是利用AuthenticationManager来进行认证了：

	final String username = serviceTicketRequest ? CAS_STATEFUL_IDENTIFIER : CAS_STATELESS_IDENTIFIER;
    String password = obtainArtifact(request);

    if (password == null) {
        logger.debug("Failed to obtain an artifact (cas ticket)");
        password = "";
    }

    final UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);

    authRequest.setDetails(authenticationDetailsSource.buildDetails(request));

    return this.getAuthenticationManager().authenticate(authRequest)

一旦使用上述CasAuthenticationProvider认证成功，则需要将认证结果Authentication保存到SecurityContext中，然后使用SecurityContextPersistenceFilter将SecurityContext保存到session中















