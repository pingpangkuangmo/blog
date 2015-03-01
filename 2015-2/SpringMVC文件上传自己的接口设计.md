#整体的包结构
首先看下整体的包的结构，如下图

![在此输入图片描述][1]

总共分成3大块，分别如下

##org.springframework.web.multipart

存放Spring定义的文件上传接口以及异常，如

-	MultipartException对用户抛出的解析异常（隐藏底层文件上传解析包所抛出的异常）

	也就指明了，这个体系下只能抛出这种类型的异常，MaxUploadSizeExceededException是MultipartException它的子类，专门用于指定文件大小限制的异常。

	用户不应该看到底层文件上传解析包所抛出的异常，底层采用的文件上传解析包在解析文件上传时也会定义自己的解析异常，这时候就需要在整合这些jar包时,需要对解析包所抛出的异常进行转换成上述已统一定义的面向用户的异常

-	MultipartFile 定义了文件解析的统一结果类型
-	MultipartResolver 定义了文件解析的处理器，不同的处理器不同的解析方式


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
		protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) 
		throws Exception {
			Object arg;
	
			HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
			MultipartHttpServletRequest multipartRequest =
				WebUtils.getNativeRequest(servletRequest, MultipartHttpServletRequest.class);
	
			if (MultipartFile.class.equals(parameter.getParameterType())) {
				assertIsMultipartRequest(servletRequest);
				Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: 
					is a MultipartResolver configured?");
				arg = multipartRequest.getFile(name);
			}
			else if (isMultipartFileCollection(parameter)) {
				assertIsMultipartRequest(servletRequest);
				Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: 
					is a MultipartResolver configured?");
				arg = multipartRequest.getFiles(name);
			}
			else if(isMultipartFileArray(parameter)) {
				assertIsMultipartRequest(servletRequest);
				Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: 
					is a MultipartResolver configured?");
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
		MultipartHttpServletRequest req = WebUtils.getNativeRequest(
			servletRequest, MultipartHttpServletRequest.class);
		if (req != null) {
			this.multipartResolver.cleanupMultipart(req);
		}
	}
	
这里其实就是调用MultipartResolver接口的void cleanupMultipart(MultipartHttpServletRequest request)方法

至此SpringMVC已经完成了自己的文件上传框架体系，即底层不管采用何种文件解析包都是走这样的一个流程。这样的一个流程其实就是对实际业务的抽象过程。我们在写代码的时候，经常就缺少抽象的能力，即很少抽象出各种业务逻辑的共同点。

#整合apache fileupload对文件上传的解析
刚才说了整个文件上传的处理流程，然后我们就来看下apache fileupload是如何整合进来的。即CommonsMultipartResolver是如何实现的

##判断一个request是否是multipart形式的
	
		@Override
		public boolean isMultipart(HttpServletRequest request) {
			return (request != null && ServletFileUpload.isMultipartContent(request));
		}

这里就是使用apache fileupload自己的ServletFileUpload.isMultipartContent判断方法，上一篇文章已经讲述了，这里不再说明了。

这里我们可以再多想一下，功能的职责划分问题（虽然问题很简单，主要是想引导大家在写代码的时候多去思考）。

因为目前判断一个request是否是multipart形式，都是一样的，不管你是哪种解析包，为什么SpringMVC不统一进行判断，而是采用解析包的判断？

如果SpringMVC自己进行统一的判断，似乎也没什么问题。站在apache fileupload的角度来说，判断request是否是multipart形式 的确应该是它的一个功能，而不是等待外界来判断。

SpringMVC既然采用第三方的解析包，就要遵守人家解析包的判断逻辑，而不是自行判断，虽然他们目前的判断逻辑是一样的。万一后来又出来一个解析包，判断逻辑不一样呢？如果流程体系还是采用SpringMVC自己的判断，可能就没法正常解析了

##将HttpServletRequest解析成DefaultMultipartHttpServletRequest

一旦上述判断通过了，则就需要执行解析过程（可以立即解析，也可以延迟解析），看下具体的解析过程

	public MultipartHttpServletRequest resolveMultipart(final HttpServletRequest request) throws MultipartException {
		Assert.notNull(request, "Request must not be null");
		if (this.resolveLazily) {
			return new DefaultMultipartHttpServletRequest(request) {
				@Override
				protected void initializeMultipart() {
					MultipartParsingResult parsingResult = parseRequest(request);
					setMultipartFiles(parsingResult.getMultipartFiles());
					setMultipartParameters(parsingResult.getMultipartParameters());
					setMultipartParameterContentTypes(parsingResult.getMultipartParameterContentTypes());
				}
			};
		}
		else {
			MultipartParsingResult parsingResult = parseRequest(request);
			return new DefaultMultipartHttpServletRequest(request, parsingResult.getMultipartFiles(),
					parsingResult.getMultipartParameters(), parsingResult.getMultipartParameterContentTypes());
		}
	}

这里大致说下过程，详细的内容去看源代码。

-	使用apache fileupload的ServletFileUpload对request进行解析，解析结果为List<FileItem>，代码如下：
	
	List<FileItem> fileItems = ((ServletFileUpload) fileUpload).parseRequest(request);

-	FileItem为apache fileupload自己的解析结果，需要转化为SpringMVC自己定义的MultipartFile

	protected MultipartParsingResult parseFileItems(List<FileItem> fileItems, String encoding) {
		MultiValueMap<String, MultipartFile> multipartFiles = new LinkedMultiValueMap<String,MultipartFile>();
		Map<String, String[]> multipartParameters = new HashMap<String, String[]>();
		Map<String, String> multipartParameterContentTypes = new HashMap<String, String>();

		// Extract multipart files and multipart parameters.
		for (FileItem fileItem : fileItems) {
			if (fileItem.isFormField()) {
				String value;
				String partEncoding = determineEncoding(fileItem.getContentType(), encoding);
				if (partEncoding != null) {
					try {
						value = fileItem.getString(partEncoding);
					}
					catch (UnsupportedEncodingException ex) {
						value = fileItem.getString();
					}
				}
				else {
					value = fileItem.getString();
				}
				String[] curParam = multipartParameters.get(fileItem.getFieldName());
				if (curParam == null) {
					// simple form field
					multipartParameters.put(fileItem.getFieldName(), new String[] {value});
				}
				else {
					// array of simple form fields
					String[] newParam = StringUtils.addStringToArray(curParam, value);
					multipartParameters.put(fileItem.getFieldName(), newParam);
				}
				multipartParameterContentTypes.put(fileItem.getFieldName(), fileItem.getContentType());
			}
			else {
				// multipart file field
				CommonsMultipartFile file = new CommonsMultipartFile(fileItem);
				multipartFiles.add(file.getName(), file);
			}
		}
		return new MultipartParsingResult(multipartFiles, multipartParameters, 
					multipartParameterContentTypes);
	}

这里有普通字段的处理和文件字段的处理。还记得上文讲的org.springframework.web.multipart.commons包的CommonsMultipartFile吗？可以看到通过new CommonsMultipartFile(fileItem)，就将FileItem结果转化为了MultipartFile结果。

至此就将HttpServletRequest解析成了DefaultMultipartHttpServletRequest，所以我们在使用request时，它的类型其实就是DefaultMultipartHttpServletRequest类型，我们可以通过它来获取各种上传的文件信息。

##清理临时文件

其实就是对所有的CommonsMultipartFile中的FileItem进行删除临时文件的操作，这个删除操作是apache fileupload自己定义的，如下
	
	protected void cleanupFileItems(MultiValueMap<String, MultipartFile> multipartFiles) {
		for (List<MultipartFile> files : multipartFiles.values()) {
			for (MultipartFile file : files) {
				if (file instanceof CommonsMultipartFile) {
					CommonsMultipartFile cmf = (CommonsMultipartFile) file;
					cmf.getFileItem().delete();
				}
			}
		}
	}

至此，SpringMVC与apache fileupload的整合完成了，其他的整合也是类似的操作。

#整合Spring自己对文件上传的解析

这个不再详细说明，主要引出来 javax.servlet.http.Part 这个对象是j2ee内置的文件上传解析结果，类似apache fileupload的FileItem解析结果，从Servlet3.0才加入进来的。

和apache fileupload一样的步骤，来看下具体源码内容：

##判断一个request是否是multipart形式的
	
	@Override
	public boolean isMultipart(HttpServletRequest request) {
		// Same check as in Commons FileUpload...
		if (!"post".equals(request.getMethod().toLowerCase())) {
			return false;
		}
		String contentType = request.getContentType();
		return (contentType != null && contentType.toLowerCase().startsWith("multipart/"));
	}

同样是这两个条件，post和"multipart/"开头。

##将HttpServletRequest解析成StandardMultipartHttpServletRequest

	@Override
	public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException {
		return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
	}

在创建StandardMultipartHttpServletRequest的时候进行解析，解析过程和apache fileupload非常类似，只不过用Part替代了apache fileupload的FileItem，如下

	private void parseRequest(HttpServletRequest request) {
		try {
			Collection<Part> parts = request.getParts();
			this.multipartParameterNames = new LinkedHashSet<String>(parts.size());
			MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<String, MultipartFile>(parts.size());
			for (Part part : parts) {
				String filename = extractFilename(part.getHeader(CONTENT_DISPOSITION));
				if (filename != null) {
					files.add(part.getName(), new StandardMultipartFile(part, filename));
				}
				else {
					this.multipartParameterNames.add(part.getName());
				}
			}
			setMultipartFiles(files);
		}
		catch (Exception ex) {
			throw new MultipartException("Could not parse multipart servlet request", ex);
		}
	}

遍历所有的Part,把每一个Part转化成StandardMultipartFile，而apache fileupload则是转化成CommonsMultipartFile。不再详细说明，具体的可以去看源码。

这里还有很多小插曲。

-	我之前导入的一直是

		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
			<version>2.5</version>
			<scope>provided</scope>
		</dependency>
	
	之后把它换成3点多的版本，还是没找到javax.servlet.http.Part，最后才发现导入的是下面的形式
	
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>javax.servlet-api</artifactId>
			<version>3.0.1</version>
			<scope>provided</scope>
		</dependency>

	这里的scope是provided，即不再加入运行环境，直接使用tomcat容器自身的servlet-api。目前我的tomcat7中servlet-api.jar包是含有这个javax.servlet.http.Part对象的，所以是可以的

-	然后我就替换掉apache fileupload，使用Servlet3自带的Part功能，来使用文件上传，发现不行，没有得到解析结果，就想尝试调试下，然而运行到Collection<Part> parts = request.getParts()这里的时候，就不能查看源文件了，这里的request是org.apache.catalina.connector.RequestFacade类型，没有关联到源文件，经过一番寻找，最终找到tomcat的maven依赖
	
		<dependency>
			<groupId>org.apache.tomcat</groupId>
			<artifactId>tomcat-catalina</artifactId>
			<version>7.0.55</version>
			<scope>provided</scope>
		</dependency>

	有了它，我们就可以在调试的时候，查看tomcat内部的运行情况了

-	然后一路跟踪，定位到结果为 需要将org.apache.catalina.core.StandardContext的allowCasualMultipartParsing属性设置为true，即允许进行文件解析,默认为false。需要在server.xml中修改工程配置,然后就大功告成了。
	
	<Context ... allowCasualMultipartParsing="true"/>

	
  [1]: http://static.oschina.net/uploads/space/2015/0228/181740_Newd_2287728.png
  [2]: http://static.oschina.net/uploads/space/2015/0216/111637_pAjl_2287728.png
