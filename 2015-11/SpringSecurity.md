参考如下文章整理的笔记：

-	[Spring Security系列](http://www.iteye.com/blogs/subjects/spring_security)

#1 Authentication认证信息

认证前：用来表示用户输入的认证信息的对象

认证后：包含用户详细信息以及权限信息的对象

接口定义如下：

	public interface Authentication extends Principal, Serializable {

		//获取用户权限
	    Collection<? extends GrantedAuthority> getAuthorities();
	
		//获取密码信息
	    Object getCredentials();
	
	    Object getDetails();

		//获取用户唯一标示，如用户名
	    Object getPrincipal();
	
	    boolean isAuthenticated();
	
	    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
	}


接口实现如下：

![Authentication接口实现](https://static.oschina.net/uploads/img/201511/02215130_oBCC.png "Authentication接口实现")


#2 SecurityContextHolder用来保存SecurityContext信息的

SecurityContextHolder默认使用ThreadLocalSecurityContextHolderStrategy策略来保存。

ThreadLocalSecurityContextHolderStrategy策略实现就是使用ThreadLocal来保存

	final class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {
  
    	private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<SecurityContext>();
	}

SecurityContext又是用来保存上述Authentication信息，接口定义如下：

	public interface SecurityContext extends Serializable {
   
	    Authentication getAuthentication();
	
	    void setAuthentication(Authentication authentication);
	}

#3 AuthenticationManager对用户输入信息进行认证

接口定义如下：

	public interface AuthenticationManager {
   
    	Authentication authenticate(Authentication authentication) throws AuthenticationException;
	}

接口实现如下：

![AuthenticationManager接口定义](https://static.oschina.net/uploads/img/201511/02215934_ElJi.png "AuthenticationManager接口定义")

默认实现ProviderManager，内部包含了一个List类型的AuthenticationProvider providers，也就是说ProviderManager委托内部的AuthenticationProvider来实现认证的

使用如下标签默认就是创建ProviderManager：

	<security:authentication-manager alias="authenticationManager">
      	<security:authentication-provider  user-service-ref="userDetailsService"/>
		<security:authentication-provider  ref="xxxxx"/>
    </security:authentication-manager>

#4 AuthenticationProvider对用户输入信息进行认证

接口的定义如下：

	public interface AuthenticationProvider {
   
	    Authentication authenticate(Authentication authentication)throws AuthenticationException;
	
	    boolean supports(Class<?> authentication);
	}

接口实现如下：

![AuthenticationProvider接口实现](https://static.oschina.net/uploads/img/201511/02220530_wBmR.png "AuthenticationProvider接口实现")

默认实现DaoAuthenticationProvider：

内部含有一个UserDetailsService userDetailsService对象，用来加载用户信息包含权限信息

-	1 使用UserDetailsService userDetailsService来加载用户信息到UserDetails对象中

		UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);

	因此在UserDetailsService上述加载过程实现中，我们就可以根据用户名来判断用户是否存在。

	UserDetails接口实现，默认就是org.springframework.security.core.userdetails.User，所以我们一般需要继承该对象

-	2 检查上述用户信息是否过期、被锁定等等，分成preAuthenticationChecks，检查密码是否正确和postAuthenticationChecks

		private class DefaultPreAuthenticationChecks implements UserDetailsChecker {
	        public void check(UserDetails user) {
	            if (!user.isAccountNonLocked()) {
	                logger.debug("User account is locked");
	
	                throw new LockedException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.locked",
	                        "User account is locked"), user);
	            }
	
	            if (!user.isEnabled()) {
	                logger.debug("User account is disabled");
	
	                throw new DisabledException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.disabled",
	                        "User is disabled"), user);
	            }
	
	            if (!user.isAccountNonExpired()) {
	                logger.debug("User account is expired");
	
	                throw new AccountExpiredException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.expired",
	                        "User account has expired"), user);
	            }
	        }
	    }
	
		protected void additionalAuthenticationChecks(UserDetails userDetails,
            UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
	        Object salt = null;
	
	        if (this.saltSource != null) {
	            salt = this.saltSource.getSalt(userDetails);
	        }
	
	        if (authentication.getCredentials() == null) {
	            logger.debug("Authentication failed: no credentials provided");
	
	            throw new BadCredentialsException(messages.getMessage(
	                    "AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"), userDetails);
	        }
	
	        String presentedPassword = authentication.getCredentials().toString();
	
	        if (!passwordEncoder.isPasswordValid(userDetails.getPassword(), presentedPassword, salt)) {
	            logger.debug("Authentication failed: password does not match stored value");
	
	            throw new BadCredentialsException(messages.getMessage(
	                    "AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"), userDetails);
	        }
	    }

	其中上述密码检查，接口是PasswordEncoder，默认是PlaintextPasswordEncoder，没有加密过的

		private class DefaultPostAuthenticationChecks implements UserDetailsChecker {
	        public void check(UserDetails user) {
	            if (!user.isCredentialsNonExpired()) {
	                logger.debug("User account credentials have expired");
	
	                throw new CredentialsExpiredException(messages.getMessage(
	                        "AbstractUserDetailsAuthenticationProvider.credentialsExpired",
	                        "User credentials have expired"), user);
	            }
	        }
	    }
	
-	3 一旦认证通过，构建一个UsernamePasswordAuthenticationToken对象，使用上述UserDetails信息、用户输入的密码信息

		protected Authentication createSuccessAuthentication(Object principal, Authentication authentication,
            UserDetails user) {
	        UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(principal,
	                authentication.getCredentials(), authoritiesMapper.mapAuthorities(user.getAuthorities()));
	        result.setDetails(authentication.getDetails());
	
	        return result;
	    }

#5 认证过程

-	1、用户使用用户名和密码进行登录。
-	2、Spring Security将获取到的用户名和密码封装成一个实现了Authentication接口的UsernamePasswordAuthenticationToken。
-	3、将上述产生的token对象传递给AuthenticationManager进行登录认证。
-	4、AuthenticationManager认证成功后将会返回一个封装了用户权限等信息的Authentication对象。
-	5、通过调用SecurityContextHolder.getContext().setAuthentication(...)将AuthenticationManager返回的Authentication对象赋予给当前的SecurityContext
		
ExceptionTranslationFilter是用来处理来自AbstractSecurityInterceptor抛出的AuthenticationException和AccessDeniedException的

	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        try {
            chain.doFilter(request, response);

            logger.debug("Chain processed normally");
        }
        catch (IOException ex) {
            throw ex;
        }
        catch (Exception ex) {
            // Try to extract a SpringSecurityException from the stacktrace
            Throwable[] causeChain = throwableAnalyzer.determineCauseChain(ex);
            RuntimeException ase = (AuthenticationException)
                    throwableAnalyzer.getFirstThrowableOfType(AuthenticationException.class, causeChain);

            if (ase == null) {
                ase = (AccessDeniedException)throwableAnalyzer.getFirstThrowableOfType(AccessDeniedException.class, causeChain);
            }

            if (ase != null) {
                handleSpringSecurityException(request, response, chain, ase);
            } else {
                // Rethrow ServletExceptions and RuntimeExceptions as-is
                if (ex instanceof ServletException) {
                    throw (ServletException) ex;
                }
                else if (ex instanceof RuntimeException) {
                    throw (RuntimeException) ex;
                }

                // Wrap other Exceptions. This shouldn't actually happen
                // as we've already covered all the possibilities for doFilter
                throw new RuntimeException(ex);
            }
        }
    }

AbstractSecurityInterceptor是Spring Security用于拦截请求进行权限鉴定的，其拥有两个具体的子类，拦截方法调用的MethodSecurityInterceptor和拦截URL请求的FilterSecurityInterceptor。

当ExceptionTranslationFilter捕获到的是AuthenticationException时将调用AuthenticationEntryPoint引导用户进行登录；

如果捕获的是AccessDeniedException，但是用户还没有通过认证，则调用AuthenticationEntryPoint引导用户进行登录认证，否则将返回一个表示不存在对应权限的403错误码



-	1 AbstractSecurityInterceptor对用户访问资源进行检查，如果未认证或者没权限则抛出异常
-	2 ExceptionTranslationFilter这个Filter捕获上述异常信息，引导重定向到登陆界面
-	3 用户输入用户名和密码进行认证登陆
-	4 Spring Security将获取到的用户名和密码封装成一个实现了Authentication接口的UsernamePasswordAuthenticationToken
-	5 将上述产生的token对象传递给AuthenticationManager进行登录认证。
-	6 AuthenticationManager认证成功后将会返回一个封装了用户权限等信息的Authentication对象
-	7 将上述Authentication对象封装到SecurityContext中
-	8 SecurityContextPersistentFilter将上述SecurityContext保存到对应的HttpSession属性中，key为：SPRING_SECURITY_CONTEXT

	SecurityContextPersistentFilter在Filter执行前，从HttpSession中先尝试获取保存的SecurityContext对象信息，如果没有则创建一个。如果HttpSession没有的话，看SecurityContextPersistentFilter的forceEagerSessionCreation属性，如果强制产生，则在此处调用request的getSession方法产生HttpSession

	Filter执行后，会将上述SecurityContext对象信息保存到HttpSession的属性中

	因此，同样用户再次访问的时候，就不需要进行登录认证了，只需要从HttpSession中就能取出对应的认证信息SecurityContext了

	也就是说，用户在某一次请求的过程中，SecurityContext是保存在ThreadLocal中的，但是请求结束后，就会被清除了。同时，SecurityContext也被保存在HttpSession中，这样的话，同一个用户不同请求即使是在不同的线程上，也都能根据session来获取到SecurityContext信息。

	![SecurityContextPersistentFilter执行逻辑](https://static.oschina.net/uploads/img/201511/02233236_mdz1.png "SecurityContextPersistentFilter执行逻辑")

#6 Filter链

Spring Security已经定义了一些Filter，不管实际应用中你用到了哪些，它们应当保持如下顺序。

-	（1）ChannelProcessingFilter，如果你访问的channel错了，那首先就会在channel之间进行跳转，如http变为https。
-	（2）SecurityContextPersistenceFilter，这样的话在一开始进行request的时候就可以在SecurityContextHolder中建立一个SecurityContext，然后在请求结束的时候，任何对SecurityContext的改变都可以被copy到HttpSession。
-	（3）ConcurrentSessionFilter，因为它需要使用SecurityContextHolder的功能，而且更新对应session的最后更新时间，以及通过SessionRegistry获取当前的SessionInformation以检查当前的session是否已经过期，过期则会调用LogoutHandler。
-	（4）认证处理机制，如UsernamePasswordAuthenticationFilter，CasAuthenticationFilter，BasicAuthenticationFilter等，以至于SecurityContextHolder可以被更新为包含一个有效的Authentication请求。
-	（5）SecurityContextHolderAwareRequestFilter，它将会把HttpServletRequest封装成一个继承自HttpServletRequestWrapper的SecurityContextHolderAwareRequestWrapper，同时使用SecurityContext实现了HttpServletRequest中与安全相关的方法。
-	（6）JaasApiIntegrationFilter，如果SecurityContextHolder中拥有的Authentication是一个JaasAuthenticationToken，那么该Filter将使用包含在JaasAuthenticationToken中的Subject继续执行FilterChain。
-	（7）RememberMeAuthenticationFilter，如果之前的认证处理机制没有更新SecurityContextHolder，并且用户请求包含了一个Remember-Me对应的cookie，那么一个对应的Authentication将会设给SecurityContextHolder。
-	（8）AnonymousAuthenticationFilter，如果之前的认证机制都没有更新SecurityContextHolder拥有的Authentication，那么一个AnonymousAuthenticationToken将会设给SecurityContextHolder。
-	（9）ExceptionTransactionFilter，用于处理在FilterChain范围内抛出的AccessDeniedException和AuthenticationException，并把它们转换为对应的Http错误码返回或者对应的页面。
-	（10）FilterSecurityInterceptor，保护Web URI，并且在访问被拒绝时抛出异常


在web.xml中如下配置：

		<filter>
	      <filter-name>springSecurityFilterChain</filter-name>
	     <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
	   </filter>
	   <filter-mapping>
	      <filter-name>springSecurityFilterChain</filter-name>
	      <url-pattern>/*</url-pattern>
	   </filter-mapping>

DelegatingFilterProxy其实是委托给Spring容器中的Filter来执行，如何指定哪一个呢？采用DelegatingFilterProxy自身的targetBeanName这个参数

上述就是默认取值spring容器中的这个springSecurityFilterChain Filter，采用的是FilterChainProxy类

当我们使用基于Spring Security的NameSpace进行配置时，系统会自动为我们注册一个名为springSecurityFilterChain类型为FilterChainProxy的bean。该FilterChainProxy含有一个List<SecurityFilterChain> filterChains属性，我们如下配置


Spring security允许我们在配置文件中配置多个http元素，以针对不同形式的URL使用不同的安全控制。Spring Security将会为每一个http元素创建对应的FilterChain，同时按照它们的声明顺序加入到FilterChainProxy。所以当我们同时定义多个http元素时要确保将更具有特性的URL配置在前。

	   <security:http pattern="/login*.jsp*" security="none"/>
	   <!-- http元素的pattern属性指定当前的http对应的FilterChain将匹配哪些URL，如未指定将匹配所有的请求 -->
	   <security:http pattern="/admin/**">
	      <security:intercept-url pattern="/**" access="ROLE_ADMIN"/>
	   </security:http>
	   <security:http>
	      <security:intercept-url pattern="/**" access="ROLE_USER"/>
	   </security:http>

每一个<security:http>都将对应一个SecurityFilterChain。每一个SecurityFilterChain中都包含各自的Filter

	private void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        FirewalledRequest fwRequest = firewall.getFirewalledRequest((HttpServletRequest) request);
        HttpServletResponse fwResponse = firewall.getFirewalledResponse((HttpServletResponse) response);

        List<Filter> filters = getFilters(fwRequest);

        if (filters == null || filters.size() == 0) {
            if (logger.isDebugEnabled()) {
                logger.debug(UrlUtils.buildRequestUrl(fwRequest) +
                        (filters == null ? " has no matching filters" : " has an empty filter list"));
            }

            fwRequest.reset();

            chain.doFilter(fwRequest, fwResponse);

            return;
        }

        VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
        vfc.doFilter(fwRequest, fwResponse);
    }

-	1 获取该request访问的url获取匹配的<security:http>，即获取匹配的SecurityFilterChain，然后返回该SecurityFilterChain的所有Filter
-	2 上述Filter构建成一个FilterChain链
-	3 执行上述FilterChain链

默认注册了如下Filter

-	SecurityContextPersistenceFilter

	在一开始进行request的时候就可以在SecurityContextHolder中建立一个SecurityContext，然后在请求结束的时候，任何对SecurityContext的改变都可以被copy到HttpSession

-	LogoutFilter

	检查是否是登出地址

-	UsernamePasswordAuthenticationFilter

	UsernamePasswordAuthenticationFilter会检查请求地址是不是配置的登陆地址，如果是则进行认证检查，从request中获取username和password构建成UsernamePasswordAuthenticationToken，然后使用AuthenticationManager authenticationManager来进行认证过程，认证成功之后重定向到target地址

-	BasicAuthenticationFilter
-	RequestCacheAwareFilter

	session中还可以保存着key为SPRING_SECURITY_SAVED_REQUEST的类型为DefaultSavedRequest的request信息，每次请求都要从request中获取session，如果没有则不会创建session，如果有session，则从session中获取DefaultSavedRequest的request信息，如果没有保存的话，则仍然为null，默认情况下，这个filter不起作用，因为没有向session中保存上述信息

-	SecurityContextHolderAwareRequestFilter
-	AnonymousAuthenticationFilter

	如果当前用户的SecurityContext中的认证信息为null的话，创建一个默认的角色

-	SessionManagementFilter
-	ExceptionTranslationFilter

	捕获之后程序执行中的AuthenticationException和AccessDeniedException。如果用户没有认证信息，则引导用户重定向到登陆界面，如果已经有登陆信息，则认为是权限不足，重定向到权限不足界面

-	FilterSecurityInterceptor

	保护每一个受访问的uri，一旦用户权限不足，则抛出AccessDeniedException异常

我们可以看到整个过程与session打交道的时候仅仅是在SecurityContextPersistenceFilter中，请求完成之后，会将SecurityContext信息保存到session中而已。供下次使用的时候，直接从session中获取SecurityContext信息，此时的SecurityContext信息中含有用户的认证信息，因此访问一个资源的话，上述filter都不会拦截，在FilterSecurityInterceptor中也会被正常通过。

一旦session失效，不能从session中获取SecurityContext信息了，就会在FilterSecurityInterceptor抛出AccessDeniedException异常，然后被ExceptionTranslationFilter捕获，引导用户重定向到登陆界面

















