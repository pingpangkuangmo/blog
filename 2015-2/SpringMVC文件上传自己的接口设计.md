#整体的包结构
首先看下整体的包的结构，如下图

![在此输入图片描述][1]

总共分成3大块，分别如下

##org.springframework.web.multipart

存放Spring定义的文件上传接口以及异常，如

-	MultipartException对用户抛出的解析异常（隐藏底层解析包所抛出的异常），也就指明了，这个体系下只能抛出这种类型的异常，MaxUploadSizeExceededException是MultipartException它的子类，专门用于指定文件大小限制的异常。在这种情况下，底层采用的jar包在解析文件上传时也会定义自己的解析异常，这时候就需要在整合这些jar包时
-	MultipartFile 定义了文件解析的统一结果类型
-	MultipartResolver 定义了文件解析的处理器，用于决定采用哪种解析方式


##org.springframework.web.multipart.commons

用于整合apache fileupload的解析，对上述定义的接口进行实现，如

-	CommonsMultipartFile实现上述MultipartFile接口，即采用apache fileupload解析的结果为CommonsMultipartFile
-	CommonsMultipartResolver实现上述MultipartResolver，待会详细说明

##org.springframework.web.multipart.support

用于整合Spring自己的文件上传解析，对上述定义的接口进行实现，如

-	StandardMultipartFile实现上述MultipartFile接口，即采用这种方式解析的结果为StandardMultipartFile
-	StandardServletMultipartResolver实现上述MultipartResolver，待会详细说明

接下来详细看看这些源码内容

#SpringMVC自己的接口设计

##MultipartResolver接口的内容：

	public interface MultipartResolver {
		//判断当前的HttpServletRequest是否是文件上传类型
		boolean isMultipart(HttpServletRequest request);
		//将HttpServletRequest转化成MultipartHttpServletRequest
		MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;
		//清除产生的临时文件等
		void cleanupMultipart(MultipartHttpServletRequest request);
	}
##MultipartHttpServletRequest接口内容：

MultipartHttpServletRequest 继承了 HttpServletRequest 和 MultipartRequest，然后就具有了下面的两个主要功能

-	获取文件上传的每一部分的请求头信息
	
		HttpHeaders getRequestHeaders();
		HttpHeaders getMultipartHeaders(String paramOrFileName);

	这里的请求头信息就是如下内容中的 Content-Disposition: form-data; name="myFile"; filename="资产型号规格模板1.xlsx"
    Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet 等信息
	
	![文件上传的请求头信息][2]

-	获取文件上传的文件内容（每个文件信息都是MultipartFile类型）

		Iterator<String> getFileNames();
		MultipartFile getFile(String name);
		List<MultipartFile> getFiles(String name);
		Map<String, MultipartFile> getFileMap();

##整个处理流程

在SpringMVC的入口类DispatcherServlet中的doDispatch方法中，可以看到是如下的处理流程

	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		boolean multipartRequestParsed = false;
		try {
			//略
			//步骤一
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);
			//略
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		finally {
			//略
			// Clean up any resources used by a multipart request.
			//步骤二
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}

可以看到这里主要有两个步骤

-	步骤一  检查是否是文件上传类型，如果是则进行解析，然后将HttpServletRequest request封装成MultipartHttpServletRequest
-	步骤二  如果是文件上传，则进行资源清理，如删除上传的临时文件等

下面分别来说

###判断并解析HttpServletRequest成MultipartHttpServletRequest：

	protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
		if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
			if (request instanceof MultipartHttpServletRequest) {
				logger.debug("Request is already a MultipartHttpServletRequest - if not in a forward, " +
						"this typically results from an additional MultipartFilter in web.xml");
			}
			else if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) instanceof MultipartException) {
				logger.debug("Multipart resolution failed for current request before - " +
						"skipping re-resolution for undisturbed error rendering");
			}
			else {
				return this.multipartResolver.resolveMultipart(request);
			}
		}
		// If not returned before: return original request.
		return request;
	}

-	首先看看DispatcherServlet的multipartResolver属性是否有值，而我们在xml文件中如下的配置就是向DispatcherServlet注入multipartResolver属性
		
		<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">  
			<property name="defaultEncoding" value="UTF-8" />  
		</bean>

	DispatcherServlet在初始化的时候，会去寻找id为"multipartResolver"并且类型为MultipartResolver的bean,所以id必须为MULTIPART_RESOLVER_BEAN_NAME即"multipartResolver",如下

		private void initMultipartResolver(ApplicationContext context) {
		try {
			this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using MultipartResolver [" + this.multipartResolver + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Default is no multipart resolver.
			this.multipartResolver = null;
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate MultipartResolver with name '" + MULTIPART_RESOLVER_BEAN_NAME +
						"': no multipart request handling provided");
			}
		}
	}

-	当multipartResolver属性有值的时候，先调用它的boolean isMultipart(HttpServletRequest request)方法，判断当前的request是否是符合文件上传类型，如果符合则调用它的MultipartHttpServletRequest resolveMultipart(HttpServletRequest request)方法将当前的request进行解析并且封装成MultipartHttpServletRequest类型。有了MultipartHttpServletRequest，我们就能获取上传的文件信息了。
	
	然后我们就可以通过2中途径来获取上传的文件。

	-	途径1 直接使用MultipartHttpServletRequest request作为参数，如下
		
			@RequestMapping(value="/test/file",method=RequestMethod.POST)
			@ResponseBody
			public String fileUpload(MultipartHttpServletRequest request){
				Map<String, MultipartFile> files=request.getFileMap();
				//使用files
				return "success";
			}

	-	途径2	使用@RequestParam("myFile") 来获取文件（RequestParam里面的"myFile"是input标签的name的值而不是文件名），如下
		
			@RequestMapping(value="/test/file",method=RequestMethod.POST)
			@ResponseBody
			public String fileUpload(@RequestParam("myFile") MultipartFile file){
				//使用file
				return "success";
			}

	对于途径1很好理解，对于途径2，为什么呢？
	
	这里简单提下，对于@RequestParam注解是由RequestParamMethodArgumentResolver来进行处理的，是它进行了特殊处理，当@RequestParam修饰的类型为MultipartFile或者javax.servlet.http.Part（后面再详细说此Part）时进行特殊处理，如下

		@Override
		protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
			Object arg;
	
			HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
			MultipartHttpServletRequest multipartRequest =
				WebUtils.getNativeRequest(servletRequest, MultipartHttpServletRequest.class);
	
			if (MultipartFile.class.equals(parameter.getParameterType())) {
				assertIsMultipartRequest(servletRequest);
				Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
				arg = multipartRequest.getFile(name);
			}
			else if (isMultipartFileCollection(parameter)) {
				assertIsMultipartRequest(servletRequest);
				Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
				arg = multipartRequest.getFiles(name);
			}
			else if(isMultipartFileArray(parameter)) {
				assertIsMultipartRequest(servletRequest);
				Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
				arg = multipartRequest.getFiles(name).toArray(new MultipartFile[0]);
			}
			else if ("javax.servlet.http.Part".equals(parameter.getParameterType().getName())) {
				assertIsMultipartRequest(servletRequest);
				arg = servletRequest.getPart(name);
			}
			else if (isPartCollection(parameter)) {
				assertIsMultipartRequest(servletRequest);
				arg = new ArrayList<Object>(servletRequest.getParts());
			}
			else if (isPartArray(parameter)) {
				assertIsMultipartRequest(servletRequest);
				arg = RequestPartResolver.resolvePart(servletRequest);
			}
			else {
				arg = null;
				if (multipartRequest != null) {
					List<MultipartFile> files = multipartRequest.getFiles(name);
					if (!files.isEmpty()) {
						arg = (files.size() == 1 ? files.get(0) : files);
					}
				}
				if (arg == null) {
					String[] paramValues = webRequest.getParameterValues(name);
					if (paramValues != null) {
						arg = paramValues.length == 1 ? paramValues[0] : paramValues;
					}
				}
			}
	
			return arg;
		}

	我们这里可以看到，其实也是通过MultipartHttpServletRequest的getFile等方法来获取的，同时支持数组、集合形式的参数


###清理占用的资源，如临时文件

	protected void cleanupMultipart(HttpServletRequest servletRequest) {
		MultipartHttpServletRequest req = WebUtils.getNativeRequest(servletRequest, MultipartHttpServletRequest.class);
		if (req != null) {
			this.multipartResolver.cleanupMultipart(req);
		}
	}
	
这里其实就是调用MultipartResolver的void cleanupMultipart(MultipartHttpServletRequest request)方法

#整合apache fileupload对文件上传的解析

#整合Spring自己对文件上传的解析


  [1]: http://static.oschina.net/uploads/space/2015/0228/181740_Newd_2287728.png
  [2]: http://static.oschina.net/uploads/space/2015/0216/111637_pAjl_2287728.png
