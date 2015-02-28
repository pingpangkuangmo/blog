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

-	获取文件上传的文件信息（每个文件信息都是MultipartFile类型）

		Iterator<String> getFileNames();
		MultipartFile getFile(String name);
		List<MultipartFile> getFiles(String name);
		Map<String, MultipartFile> getFileMap();

##整个处理流程



#整合apache fileupload对文件上传的解析

#整合Spring自己对文件上传的解析


  [1]: http://static.oschina.net/uploads/space/2015/0228/181740_Newd_2287728.png