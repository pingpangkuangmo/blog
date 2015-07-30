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

下面就通过源码来详细说明这个建立连接的过程。

#3 客户端的建立连接的过程

##3.1 大体连接过程概述

首先与ZooKeeper服务器建立连接，有两层连接要建立。

-	客户端与服务器端的TCP连接
-	在TCP连接的基础上建立session关联

建立TCP连接之后，客户端发送ConnectRequest请求，申请建立session关联，此时服务器端会为该客户端分配sessionId和密码，同时开启对该session是否超时的检测。

当在sessionTimeout时间内，即还未超时，此时TCP连接断开，服务器端仍然认为该sessionId处于存活状态。此时，客户端会选择下一个ZooKeeper服务器地址进行TCP连接建立，TCP连接建立完成后，拿着之前的sessionId和密码发送ConnectRequest请求，如果还未到该sessionId的超时时间，则表示自动重连成功，对客户端用户是透明的，一切都在背后默默执行，ZooKeeper对象是有效的。

如果重新建立TCP连接后，已经达到该sessionId的超时时间了（服务器端就会清理与该sessionId相关的数据），则会报session过期异常，此时ZooKeeper对象就是无效的了，必须要重新new一个新的ZooKeeper对象，分配新的sessionId了。


##3.2 ZooKeeper对象

它是面向用户的，提供一些操作API。

它又两个重要的属性：

-	ClientCnxn cnxn：负责所有的ZooKeeper节点操作的执行
-	ZKWatchManager watchManager：负责维护某个path上注册的Watcher

如创建某个node操作（同步方式）：

ZooKeeper对象负责创建出Request，并交给ClientCnxn来执行，ZooKeeper对象再对返回结果进行处理。

![ZooKeeper同步方式创建节点操作](https://static.oschina.net/uploads/img/201507/29192425_UnoE.png "ZooKeeper同步方式创建节点操作")

下面来看下异步回调的方式创建node：

![ZooKeeper异步方式创建节点操作](https://static.oschina.net/uploads/img/201507/29193916_3lkA.png "ZooKeeper异步方式创建节点操作")

对于两者的实现细节后文再详细说明。

至此简单了解了，ZooKeeper对象主要封装用户的请求以及处理响应等操作。用户请求的执行全部交给ClientCnxn来执行，那我们就详细看下ClientCnxn的来源及大体内容。

先看看ClientCnxn是怎么来的：

![输入图片说明](https://static.oschina.net/uploads/img/201507/29195113_Gcjr.png "在这里输入图片标题")

-	第一步：为ZKWatchManager watchManager设置一个默认的Watcher
-	第二步：将连接字符串信息交给ConnectStringParser进行解析

	连接字符串比如： "192.168.12.1:2181,192.168.12.2:2181,192.168.12.3:2181/root"

	解析结果如下：

	![ConnectStringParser解析结果](https://static.oschina.net/uploads/img/201507/29200845_MSRF.png "ConnectStringParser解析结果")

	得到两个数据String chrootPath默认的跟路径和ArrayList<InetSocketAddress> serverAddresses即多个host和port信息。

-	第三步：根据上述解析的host和port列表结果，创建一个HostProvider

	有了ConnectStringParser的解析结果，为什么还需要一个HostProvider再来包装下呢？主要是为将来留下扩展的余地

	来看下HostProvider的详细接口介绍：

	![HostProvider的详细接口介绍](https://static.oschina.net/uploads/img/201507/29204007_OfNi.png "HostProvider的详细接口介绍")

	HostProvider主要负责不断的对外提供可用的ZooKeeper服务器地址，这些服务器地址可以是从一个url中加载得来或者其他途径得来。同时对于不同的ZooKeeper客户端，给出就近的ZooKeeper服务器地址等。

	来看下默认的HostProvider实现StaticHostProvider：

	![StaticHostProvider](https://static.oschina.net/uploads/img/201507/29205354_YYaY.png "StaticHostProvider")

	有三个属性，一个就是服务器地址列表（经过如下方式随机打乱了）：

		Collections.shuffle(this.serverAddresses)

	另外两个属性用于标记，下面来具体看下，StaticHostProvider是如何实现不断的对外提供ZooKeeper服务器地址的：

	![StaticHostProvider提供ZooKeeper服务器地址](https://static.oschina.net/uploads/img/201507/29210008_UvTp.png "StaticHostProvider提供ZooKeeper服务器地址")

	代码也很简单，就是在打乱的服务器地址列表中，不断地遍历，到头之后，在从0开始。

	上面的spinDelay是个什么情况呢？

	正常情况下，currentIndex先加1，然后返回currentIndex+1的地址，当该地址连接成功后会执行onConnected方法，即lastIndex = currentIndex了。然而当返回的currentIndex+1的地址连接不成功，继续尝试下一个，仍不成功，仍继续下一个，就会遇到currentIndex=lastIndex的情况，此时即轮询了一遍，仍然没有一个地址能够连接上，此时的策略就是先暂停休息休息，然后再继续。

	
-	第四步：为创建ClientCnxn准备参数并创建ClientCnxn。

	首先是通过getClientCnxnSocket()获取一个ClientCnxnSocket。来看下ClientCnxnSocket是主要做什么工作的：

	>A ClientCnxnSocket does the lower level communication with a socket implementation.
	This code has been moved out of ClientCnxn so that a Netty implementation can be provided as an alternative to the NIO socket code.

	专门用于负责socket通信的，把一些公共部分抽象出来，其他的留给不同的实现者来实现。如可以选择默认的ClientCnxnSocketNIO，也可以使用netty等。

	来看下getClientCnxnSocket()的获取ClientCnxnSocket的过程：

	![getClientCnxnSocket过程](https://static.oschina.net/uploads/img/201507/30070323_wIkK.png "getClientCnxnSocket过程")

	首先获取系统参数"zookeeper.clientCnxnSocket",如果没有的话，使用默认的ClientCnxnSocketNIO，所以我们可以通过指定该参数来替换默认的实现。

	参数准备好了，ClientCnxn是如何来创建的呢？

	![ClientCnxn的创建过程](https://static.oschina.net/uploads/img/201507/30070937_Arhf.png "ClientCnxn的创建过程")

	首先就是保存一些对象参数，此时的sessionId和sessionPasswd都还没有。然后就是两个timeout参数：connectTimeout和readTimeout。在ClientCnxn的发送和接收数据的线程中，会不断的检测连接超时和读取超时，一旦出现超时，就认为服务不稳定，需要更换服务器，就会从HostProvider中获取下一个服务器地址进行连接。

	最后就是两个线程，一个事件线程即EventThread，一个发送和接收socket数据的线程即SendThread。

	事件线程EventThread呢就是从一个事件队列中不断取出事件并进行处理：

	![EventThread的工作职责](https://static.oschina.net/uploads/img/201507/30081219_ZXVp.png "EventThread的工作职责")

	看下具体的处理过程，主要分成两种情况，一种就是我们注册的watch事件，另一种就是处理异步回调函数：

	![watch处理和异步回调](https://static.oschina.net/uploads/img/201507/30081813_fuFZ.png "watch处理和异步回调")

	可以看到这里就是触发我们注册Watch的，还有触发上文提到的异步回调的情况的。

	明白了EventThread是如何来处理事件的，需要知道这些事件是如何来的：

	![EventThread添加事件](https://static.oschina.net/uploads/img/201507/30082302_vHHZ.png "EventThread添加事件")

	对外提供了三个方法来添加不同类型的事件，如SendThread线程就会调用这三个方法来添加事件。其中对于事件通知，会首先根据ZKWatchManager watchManager来获取关心该事件的所有Watcher，然后触发他们。

	再来看看SendThread的工作内容：

		sendThread = new SendThread(clientCnxnSocket);
	把传递给ClientCnxn的clientCnxnSocket，再传递给SendThread，让它服务于SendThread。

	在SendThread的run方法中，有一个while循环，不断的做着以下几件事：

	-	任务1：不断检测clientCnxnSocket是否和服务器处于连接状态，如果是未连接状态，则从hostProvider中取出一个服务器地址，使用clientCnxnSocket进行连接。和服务器建立连接成功后，开始发送ConnectRequest请求，把该请求放到outgoingQueue请求队列中，等待被发送给服务器

		![和服务器建立连接](https://static.oschina.net/uploads/img/201507/30194804_g1GU.png "和服务器建立连接")

		![建立socket连接后发送ConnectRequest请求来初始化session](https://static.oschina.net/uploads/img/201507/30195637_A8le.png "建立socket连接后发送ConnectRequest请求来初始化session")

		从上图可以看出，一旦socket连接建立完毕，就开始发送ConnectRequest请求来初始化session。
	-	任务2：检测是否超时：当处于连接状态时，检测是否读超时，当处于未连接状态时，检测是否连接超时

		![检测读超时或者socket连接超时](https://static.oschina.net/uploads/img/201507/30195926_gJB7.png "检测读超时或者socket连接超时")

		一旦超时，则抛出SessionTimeoutException，然后看下是如何处理呢？

		![异常处理](https://static.oschina.net/uploads/img/201507/30200251_dHc3.png "异常处理")

		可以看到一旦发生超时异常或者其他异常，都会进行清理，并设置连接状态为未连接，然后发送Disconnected事件。至此又会进入任务1的流程
	-	任务3：不断的发送ping通知，服务器端每接收到ping请求，就会从当前时间重新计算session过期时间，所以当客户端按照一定时间间隔不断的发送ping请求，就能保证客户端的session不会过期。发送时间间隔如下：

		![发送Ping通知的机制](https://static.oschina.net/uploads/img/201507/30202107_8Rh6.png "发送Ping通知的机制")

		clientCnxnSocket.getIdleSend()：是最后一次发送数据包的时间与当前时间的间隔，即readTimeout的时间已经过去一半多了，都没有发送数据包的话，则执行一次Ping发送。或者过去MAX_SEND_PING_INTERVAL（10s）都还没有发送数据包的话，则执行一次Ping发送。

		![Ping发送的内容](https://static.oschina.net/uploads/img/201507/30202838_P3s1.png "Ping发送的内容")
	
		ping发送的内容只有请求头OpCode.ping的标示，其他都为空。发送ping请求，也是把该请求放到outgoingQueue发送队列中，等待被执行。

	-	任务4：执行IO操作，即发送请求队列中的请求和读取服务器端的响应数据。

		![发送请求队列中的请求](https://static.oschina.net/uploads/img/201507/30204315_uaaw.png "发送请求队列中的请求")

		首先从outgoingQueue请求队列中取出第一个请求，然后进行序列化，然后使用socket进行发送。
	
		读取服务器端数据如下：

		![读取服务器端响应](https://static.oschina.net/uploads/img/201507/30204611_Apld.png "读取服务器端响应")

		分为两种：一种是读取针对ConnectRequest请求的响应，另一种就是其他响应，先暂时不说。

		先来看看针对ConnectRequest请求的响应：

		![读取ConnectResponse的内容](https://static.oschina.net/uploads/img/201507/30204946_yKYF.png "读取ConnectResponse的内容")

		首先进行反序列化，得到ConnectResponse对象，我们就可以获取到服务器端给我们客户端分配的sessionId和passwd，以及协商后的sessionTimeOut时间。

		![session获取成功后，重置参数](https://static.oschina.net/uploads/img/201507/30205409_XGGg.png "session获取成功后，重置参数")

		首选要根据协商后的sessionTimeout时间，重新计算readTimeout和connectTimeout值。然后保留和记录sessionId和passwd。最后通过EventThread发送一个SyncConnected连接成功事件。至此，TCP连接和session初始化请求都完成了，客户端的ZooKeeper对象可以正常使用了。

		至此，我们便了解客户端与服务器端建立连接的过程。

		

	

	