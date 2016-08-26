# 1 类型转换异常场景 
我们在使用Jedis的时候，经常会出现类型转换异常，有如下情况：

-	多线程环境

	Jedis是线程不安全的，如果存在多线程使用同一个Jedis，就会出现类型转换异常网上也流传着很多错误的解释，下面我们以一个案例来复现下这个问题，这个很好理解。

-	单线程环境

	即使在单线程的情况下，也是会出现类型转换异常的，下面就针对此做一个案例分析

# 2 Jedis类型转换异常案例

## 2.1 案例介绍

案例是从这里来的[Jedis returnResource使用注意事项](http://my.oschina.net/zhuguowei/blog/406807)

代码如下：

	public static void main(String[] args) throws Exception{
		Jedis jedis = new Jedis("192.168.126.131", 6379);
		System.out.println("get name=" + jedis.get("name"));
		System.out.println("Make SocketTimeoutException");
		System.in.read(); //等待制造SocketTimeoutException
		try {
		    System.out.println(jedis.get("name"));
		} catch (Exception e) {
		    e.printStackTrace();
		}
		System.out.println("Recover from SocketTimeoutException");
		Thread.sleep(50000); // 继续休眠一段时间 等待网络完全恢复
		boolean isMember = jedis.sismember("urls", "baidu");
		System.out.println("isMember " + isMember);
		jedis.close();
	}

以及包含2个阻断和解除网络通信的命令

-	阻断网络通信

		sudo iptables -A INPUT -p tcp --dport 6379 -j DROP

-	解除网络阻塞

		sudo iptables -F

案例运行过程描述：

-	1 创建Jedis，发送get命令，启动与redis的连接，连接成功后获取到响应数据
-	2 程序阻塞在System.in.read()，等待输入，此时我们需要将网络连接阻塞，执行上述阻断网络命令
-	3 输入任意数据，让程序不再阻塞，继续走下去，执行get命令，此时由于网络不通，导致出现SocketTimeoutException异常
-	4 打印出异常，继续往下走，sleep 50s，此时我们需要解除网络阻塞，执行上述对应命令
-	5 50s过完，就会执行jedis的sismember方法，此时就会出现类型转换异常

## 2.2 Jedis原理介绍

这里不再详细介绍。

Jedis内部有一个Socket与redis服务器建立连接。在创建Jedis对象的时候，并没有去建立连接，而是在执行命令的时候才会先检查是否已连接，未连接的话，才建立连接。

Socket一旦连接建立，就会获取到Socket的OutputStream，并用RedisOutputStream进行包装，获取到Socket的InputStream，并用RedisInputStream进行包装。RedisOutputStream内部含有一个byte buf[]数组。

也就是说在jedis在向OutputStream写入命令的时候，会先写入到上述buf数组中，然后在读取的时候，才会flush上述数据，将数据写入到Socket的OutputStream中，并调用flush，以Jedis的get方法为例

	public String get(final String key) {
		checkIsInMulti();
		client.sendCommand(Protocol.Command.GET, key);
		return client.getBulkReply();
	}

client.sendCommand方法会将数据写入到RedisOutputStream内部的buf中
client.getBulkReply方法会首先执行一次flush，即将buf中数据写入到Socket的OutputStream中，并调用Socket的OutputStream的flush。

## 2.3 类型转换异常的原因

网上很多人说造成上述场景的类型转换异常是因为：

出现SocketTimeoutException异常后，RedisOutputStream的buf中残留上次命令，没做清理处理，导致再执行其他命令时连同之前的命令一起发送过去了。

经过查看RedisOutputStream的源码，buf中确实不会去主动清除原有数据，而是每次都是直接覆盖，有count指针来标记，但是这也不会造成上述所说的影响，RedisOutputStream是OK的。

首先我们要明白什么是SocketTimeoutException异常：
上述Jedis的Socket在发送完成数据后，就会去执行读取数据，即读取Socket的InputStream中的数据，并且又一定的阻塞时间，如果redis服务器迟迟不返回数据，一旦超过SO_TIMEOUT（即Socket的读取超时时间），客户端就会抛出一个SocketTimeoutException异常。

造成这种异常的原因有很多：

-	网络闪断（会TCP重传）：上述案例情景就是网络断开，数据包发送失败，会TCP重传
-	网络没有断，但是传输比较慢，或者redis服务器处理很慢

上述原因都会造成客户端读取超时。一旦超时，我们的Jedis程序抛出异常，继续往下走，如果此时再次执行其他命令的话，仍然会读取服务器端响应，此时读到的响应就是上次请求的响应了，所以会导致类型转换异常。如果与上次请求的类型一致，那就更可怕了，错误就会被深深的掩盖过去了。

# 3 Jedis类型转换异常的解决办法

上述问题就是：我们没有正确对待这个SocketTimeoutException异常，即一旦出现SocketTimeoutException异常，我们是必须要废弃掉这个Jedis的。所以对于单线程环境下的Jedis来说，一旦出现这种异常，我们需要重新new一个新的Jedis来使用。

Jedis在内部执行出现异常，如SocketTimeoutException异常的时候，会标记一个boolean broken=true，即意味着该连接已经废弃了。

重要的大坑在这里，我们通常使用JedisPool来应对多线程环境下Jedis的使用，一般使用方式如下：

	Jedis jedis = null;//从pool中获取资源  
	try{
		jedis = pool.getResource();
	    jedis.set("k1", "v1");  
	}catch(Exception e){  
	    e.printStackTrace();
	}finally{
		if(jedis != null){
			pool.returnResource(jedis);//向连接池“归还”资源，千万不要忘记。
		}
	}

而对于JedisPool，我们会使用returnResource方法来向pool中释放回Jedis，而这个returnResource却忽视了上述boolean broken属性，直接将一个标记废弃的连接放回到了pool中，下次别人取的时候，必然出问题。

所以针对JedisPool这种情况，解决办法如下：

-	1 在上述catch中捕获SocketTimeoutException异常，调用pool的returnBrokenResource方法来释放Jedis（该方法会将Jedis实例标记为下线，无法被他人获取到了），但是不推荐这种，还要考虑其他异常等等

-	2 另一个就是直接调用Jedis的close方法，最新版2.9.0（其他版本没验证）中close方法对上述boolean broken标记进行了处理，并且将returnResource标记成废弃了，处理如下

		public void close() {
			if (dataSource != null) {
			  if (client.isBroken()) {
			    this.dataSource.returnBrokenResource(this);
			  } else {
			    this.dataSource.returnResource(this);
			  }
			} else {
			  client.close();
			}
		}

上述this.dataSource可以理解为JedisPool。
即一旦是broken，则调用pool的returnBrokenResource方法，否则调用pool的returnResource方法。

所以最终写法应该如下：

	Jedis jedis = null;//从pool中获取资源  
	try{
		jedis = pool.getResource();
	    jedis.set("k1", "v1");  
	}finally{
		if(jedis != null){
			jedis.close();
		}
	}

# 4 问题深思

可以想到2方面的问题：

问题1：jedis为什么要暴漏这么个危险的API给用户使用（即要求用户自觉的close，不自觉后果自负）

如果是我们在开发框架给被人使用，那就要尽量避免这种API的设计，把close自动隐藏在框架内部，避免了使用人员的误使用，同时减少了代码的复杂度，即使是上述最终的写法也是很丑陋的，要完成一个set功能，要关注太多地方了，这部分完全可以框架底层包装起来，只给用户一个set方法即可。


问题2：请求和响应的不匹配问题

这种不匹配的问题在同步和异步的时候分别怎么处理？

同步通信：在设计的时候，必须发送一次请求就要读取一次响应，通过这种方式来匹配。然而在某些情况下，读取响应有一定的超时时间，一旦超时，就抛出SocketTimeoutException异常，从而结束本次读取，而响应可能后来又到达了，这种情况就会造成不匹配的现象。要避免这种情况，就必须要废弃掉这个Socket了，所以如果客户端设计成同步通信的时候，一旦遇到这种异常，则就需要废弃了，重新建立连接了。

异步通信：在设计的时候一般会为每个请求分配一个请求id，服务器端在处理请求后，会把这个请求id返回给客户端，客户端根据返回的请求id来匹配是那一次的请求对应的响应，就不会出现上述那种匹配错乱的问题。异步通信在读取数据的时候也通常是有数据可读才会去执行读操作，可以减少同步通信中因网络拥堵或其他原因造成的SocketTimeoutException问题。异步通信好处的代价就是比同步通信复杂。

所以如果我们在设计的时候，就需要去考虑这样的问题，避免造出一个大坑来。