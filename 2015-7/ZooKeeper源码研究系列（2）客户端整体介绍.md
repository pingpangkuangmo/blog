#1 系列目录

-	[ZooKeeper源码环境搭建]()

#2 客户端API简单使用

##2.1 demo案例1

一个最简单的demo如下：

	public class ZookeeperConstructorSimple implements Watcher{
	
		private static CountDownLatch connectedSemaphone=new CountDownLatch(1);
	
		public static void main(String[] args) throws IOException {
			ZooKeeper zooKeeper=new ZooKeeper("127.0.0.1:2181",5000,new ZookeeperConstructorSimple());
			System.out.println(zooKeeper.getState());
			try {
				connectedSemaphone.await();
			} catch (Exception e) {}
			System.out.println("ZooKeeper session established");
			System.out.println("sessionId="+zooKeeper.getSessionId());
			System.out.println("password="+zooKeeper.getSessionPasswd());
		}
	
		@Override
		public void process(WatchedEvent event) {
			System.out.println("my ZookeeperConstructorSimple watcher Receive watched event:"+event);
			if(KeeperState.SyncConnected==event.getState()){
				connectedSemaphone.countDown();
			}
		}

	}

使用的maven依赖如下：

	<dependency>
		<groupId>org.apache.zookeeper</groupId>
		<artifactId>zookeeper</artifactId>
		<version>3.4.6</version>
	</dependency>

对于目前来说，ZooKeeper的服务器端代码和客户端代码还是混在一起的，估计日后能改吧。

使用的ZooKeeper的构造函数有三个参数构成

-	ZooKeeper集群的服务器地址列表

	该地址是可以填写多个的，以逗号分隔。如"127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183",那客户端连接的时候到底是使用哪一个呢？先随机打乱，然后轮询着用，后面再详细介绍。
-	sessionTimeout

	客户端指定的session超时时间，即超过sessionTimeout时间后，客户端没有向服务器端发送任何请求（正常情况下客户端会每隔一段时间发送心跳请求，此时服务器端会从新计算客户端的超时时间点的），则服务器端认为session超时，清理数据。此时客户端的ZooKeeper对象就不再起作用了，需要再重新new一个新的对象了。
-	Watcher

	作为ZooKeeper对象一个默认的Watcher，用于接收一些事件通知。如和服务器连接成功的通知、断开连接的通知、Session过期的通知等。


同时我们可以看到，一旦和ZooKeeper服务器连接建立成功，就会获取服务器端分配的sessionId和password，如下：

	sessionId=94249128002584594
	password=[B@4de3aaf6

下面就通过源码来详细说明这个建立连接的过程。、

#3 客户端的建立连接的过程

首先与ZooKeeper服务器建立连接，有两层连接要建立。


