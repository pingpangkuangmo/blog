#文件上传格式
先来看下含有文件上传时的表单提交是怎样的格式

	<form action="/upload/request" method="POST" enctype="multipart/form-data" id="requestForm">
		<input type="file" name="myFile">
		<input type="text" name="user">
		<input type="text" name="password">
		<input type="submit" value="提交">
	</form>


form表单提交内容如下

![chrome文件上传][1]

从上面可以看到，含有文件上传的格式是这样组织的。
	
-	文件类型字段

		------WebKitFormBoundaryCvop2jTxU5F6lj6G（分隔符）
		Content-Disposition: form-data; name="myFile"; filename="资产型号规格模板1.xlsx"
		Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
		（换行）
		（文件内容）

-	其他类型字段

		------WebKitFormBoundaryCvop2jTxU5F6lj6G（分隔符）
		Content-Disposition: form-data; name="user"
		（换行）
		lg
-	结束
	
		------WebKitFormBoundaryCvop2jTxU5F6lj6G--（分隔符加上--）

对于上面的文件内容，chrome浏览器是不显示的，换成firefox可以看到，如下图所示

![chrome文件上传][2]

同时我们还可以注意到，不同的浏览器，分隔符是不一样的，在请求头

	Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryCvop2jTxU5F6lj6G

中指明了分隔符的内容。

#文件上传注意点：	

-	一定是post提交，如果换成get提交，则浏览器默认仅仅把文件名作为属性值来上传，不会上传文件内容，如下

	![文件上传get方式][3]

-	form表单中一定不要忘了添加
		
		enctype="multipart/form-data"
	
	否则的话，浏览器则不是按照上述的格式来来传递数据的。

上述两点才能保证浏览器正常的进行文件上传。

#apache fileupload的解析

有了上述文件上传的组织格式，我们就需要合理的设计后台的解析方式，下面来看下apache fileupload的使用。先来看下整体的流程图
![apache fileUpload整体流程图][4]


##Servlets and Portlets

apache fileupload分Servlets and Portlets两种情形来处理。Servlet我们很熟悉，而Portlets我也没用过，可自行去搜索。

###判断request是否是Multipart

对于HttpServletRequest来说，另一个不再说明，自行查看源码，判断规则如下：

-	是否是post请求
-	contentType是否以multipart/开头

见源码：

	public static final boolean isMultipartContent(
            HttpServletRequest request) {
        if (!POST_METHOD.equalsIgnoreCase(request.getMethod())) {
            return false;
        }
        return FileUploadBase.isMultipartContent(new ServletRequestContext(request));
    }
	public static final boolean isMultipartContent(RequestContext ctx) {
        String contentType = ctx.getContentType();
        if (contentType == null) {
            return false;
        }
        if (contentType.toLowerCase(Locale.ENGLISH).startsWith(MULTIPART)) {
            return true;
        }
        return false;
    }

###对request进行封装

servlet的输入参数为HttpServletRequest，Portlets的输入参数为ActionRequest，数据来源不同，为了统一方便后面的数据处理，引入了RequestContext接口，来统一一下目标数据的获取，接口类图如下
![apache fileUpload整体流程图][5]


此时RequestContext就作为了数据源，不再与HttpServletRequest和ActionRequest打交道。

上述的实现过程是由FileUpload的子类ServletFileUpload和PortletFileUpload分别完成包装的，他们的类图如下
![apache fileUpload整体流程图][5]

源码展示如下：

-	ServletFileUpload类
	
		public List<FileItem> parseRequest(HttpServletRequest request)
    		throws FileUploadException {
        	return parseRequest(new ServletRequestContext(request));
    	}

-	PortletFileUpload类

		public List<FileItem> parseRequest(ActionRequest request)
            throws FileUploadException {
        	return parseRequest(new PortletRequestContext(request));
    	}

###由RequestContext数据源得到解析后的数据集合 FileItemIterator


FileItemIterator内容如下：
	
	public interface FileItemIterator {
	    boolean hasNext() throws FileUploadException, IOException;
	    FileItemStream next() throws FileUploadException, IOException;
	}

这就是一个轮询器，可以看成是FileItemStream的集合。而FileItemStream则是之前上传文件格式内容

	------WebKitFormBoundary77tsMdWQBKrQOSsV
	Content-Disposition: form-data; name="user"

	lg

或者

	------WebKitFormBoundary77tsMdWQBKrQOSsV
	Content-Disposition: form-data; name="myFile"; filename="萌芽.jpg"
	Content-Type: image/jpeg

	(文件内容)

的封装

FileItemStream内容如下：

	public interface FileItemStream extends FileItemHeadersSupport {
		/*流中包含了数值或者文件的内容*/
	    InputStream openStream() throws IOException;
	    String getContentType();
		/*用来存放文件名，不是文件字段则为null*/
	    String getName();
		/*对应input标签中的name属性*/
	    String getFieldName();
		/*标识该字段是否是一般的form字段还是文件字段*/
	    boolean isFormField();
	}

然后我们来具体看下由RequestContext如何解析成一个FileItemIterator的：




  [1]: http://static.oschina.net/uploads/space/2015/0216/111637_pAjl_2287728.png
  [2]: http://static.oschina.net/uploads/space/2015/0216/112841_9nQm_2287728.png
  [3]: http://static.oschina.net/uploads/space/2015/0216/121610_rQpt_2287728.png
  [4]: http://static.oschina.net/uploads/space/2015/0216/172522_DESE_2287728.png
  [5]: http://static.oschina.net/uploads/space/2015/0216/172522_DESE_2287728.png