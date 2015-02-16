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


  [1]: http://static.oschina.net/uploads/space/2015/0216/111637_pAjl_2287728.png
  [2]: http://static.oschina.net/uploads/space/2015/0216/112841_9nQm_2287728.png
  [3]: http://static.oschina.net/uploads/space/2015/0216/121610_rQpt_2287728.png