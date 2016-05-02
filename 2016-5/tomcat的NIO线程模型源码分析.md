#1 tomcat8的并发参数控制

这种问题其实到官方文档上查看一番就可以知道,tomcat很早的版本还是使用的BIO，之后就支持NIO了，具体版本我也不记得了，有兴趣的自己可以去查下。本篇的tomcat版本是tomcat8.5。可以到这里看下[tomcat8.5的配置参数](https://tomcat.apache.org/tomcat-8.5-doc/config/http.html)

我们先来简单回顾下目前一般的NIO服务器端的大致实现，借鉴infoq上的一篇文章[Netty系列之Netty线程模型](http://www.infoq.com/cn/articles/netty-threading-model)中的一张图

![一般NIO线程模型](https://static.oschina.net/uploads/img/201510/20083315_ObVg.png "一般NIO线程模型")

-	一个或多个Acceptor线程，每个线程都有自己的Selector，Acceptor只负责accept新的连接，一旦连接建立之后就将连接注册到其他Worker线程中

-	多个Worker线程，有时候也叫IO线程，就是专门负责IO读写的。一种实现方式就是像Netty一样，每个Worker线程都有自己的Selector，可以负责多个连接的IO读写事件，每个连接归属于某个线程。另一种方式实现方式就是有专门的线程负责IO事件监听，这些线程有自己的Selector，一旦监听到有IO读写事件，并不是像第一种实现方式那样（自己去执行IO操作），而是将IO操作封装成一个Runnable交给Worker线程池来执行，这种情况每个连接可能会被多个线程同时操作，相比第一种并发性提高了，但是也可能引来多线程问题，在处理上要更加谨慎些。tomcat的NIO模型就是第二种。

所以一般参数就是Acceptor线程个数，Worker线程个数。来具体看下参数

##1.1 acceptCount

文档描述为：

The maximum queue length for incoming connection requests when all possible request processing threads are in use. Any requests received when the queue is full will be refused. The default value is 100.

这个参数就立马牵涉出一块大内容：TCP三次握手的详细过程，这个之后再详细探讨。这里可以简单理解为：连接在被ServerSocketChannel accept之前就暂存在这个队列中，acceptCount就是这个队列的最大长度。ServerSocketChannel accept就是从这个队列中不断取出已经建立连接的的请求。所以当ServerSocketChannel accept取出不及时就有可能造成该队列积压，一旦满了连接就被拒绝了

##1.2 acceptorThreadCount

文档如下描述

The number of threads to be used to accept connections. Increase this value on a multi CPU machine, although you would never really need more than 2. Also, with a lot of non keep alive connections, you might want to increase this value as well. Default value is 1.

Acceptor线程只负责从上述队列中取出已经建立连接的请求。在启动的时候使用一个ServerSocketChannel监听一个连接端口如8080，可以有多个Acceptor线程并发不断调用上述ServerSocketChannel的accept方法来获取新的连接。参数acceptorThreadCount其实使用的Acceptor线程的个数。

##1.3 maxConnections

文档描述如下

The maximum number of connections that the server will accept and process at any given time. When this number has been reached, the server will accept, but not process, one further connection. This additional connection be blocked until the number of connections being processed falls below maxConnections at which point the server will start accepting and processing new connections again. Note that once the limit has been reached, the operating system may still accept connections based on the acceptCount setting. The default value varies by connector type. For NIO and NIO2 the default is 10000. For APR/native, the default is 8192.

Note that for APR/native on Windows, the configured value will be reduced to the highest multiple of 1024 that is less than or equal to maxConnections. This is done for performance reasons.
If set to a value of -1, the maxConnections feature is disabled and connections are not counted.

这里就是tomcat对于连接数的一个控制，即最大连接数限制。一旦发现当前连接数已经超过了一定的数量（NIO默认是10000），上述的Acceptor线程就被阻塞了，即不再执行ServerSocketChannel的accept方法从队列中获取已经建立的连接。但是它并不阻止新的连接的建立，新的连接的建立过程不是Acceptor控制的，Acceptor仅仅是从队列中获取新建立的连接。所以当连接数已经超过maxConnections后，仍然是可以建立新的连接的，存放在上述acceptCount大小的队列中，这个队列里面的连接没有被Acceptor获取，就处于连接建立了但是不被处理的状态。当连接数低于maxConnections之后，Acceptor线程就不再阻塞，继续调用ServerSocketChannel的accept方法从acceptCount大小的队列中继续获取新的连接，之后就开始处理这些新的连接的IO事件了

##1.4 maxThreads

文档描述如下

The maximum number of request processing threads to be created by this Connector, which therefore determines the maximum number of simultaneous requests that can be handled. If not specified, this attribute is set to 200. If an executor is associated with this connector, this attribute is ignored as the connector will execute tasks using the executor rather than an internal thread pool.

这个简单理解就算是上述worker的线程数，下面会详细的说明。他们专门用于处理IO事件，默认是200。


#2 tomcat的NioEndpoint

上面参数仅仅是简单了解了下参数配置，下面我们就来详细研究下tomcat的NIO服务器具体情况，这就要详细了解下tomcat的NioEndpoint实现了

先来借鉴看下[tomcat高并发场景下的BUG排查](https://yq.aliyun.com/articles/2889?spm=5176.team22.teamshow1.30.XRi499)中的一张图

![输入图片说明](https://static.oschina.net/uploads/img/201605/01113945_lXzx.png "在这里输入图片标题")

这张图勾画出了NioEndpoint的大致执行流程图，worker线程并没有体现出来，它是作为一个线程池不断的执行IO读写事件即SocketProcessor（一个Runnable），即这里的Poller仅仅监听Socket的IO事件，然后封装成一个个的SocketProcessor交给worker线程池来处理。下面我们来详细的介绍下NioEndpoint中的Acceptor、Poller、SocketProcessor

##2.1 Acceptor

###2.1.1 初始化过程

获取指定的Acceptor数量的线程

	protected final void startAcceptorThreads() {
        int count = getAcceptorThreadCount();
        acceptors = new Acceptor[count];

        for (int i = 0; i < count; i++) {
            acceptors[i] = createAcceptor();
            String threadName = getName() + "-Acceptor-" + i;
            acceptors[i].setThreadName(threadName);
            Thread t = new Thread(acceptors[i], threadName);
            t.setPriority(getAcceptorThreadPriority());
            t.setDaemon(getDaemon());
            t.start();
        }
    }

###2.1.2 Acceptor的run方法

	protected class Acceptor extends AbstractEndpoint.Acceptor {

        @Override
        public void run() {

            int errorDelay = 0;

            // Loop until we receive a shutdown command
            while (running) {

                // Loop if endpoint is paused
                while (paused && running) {
                    state = AcceptorState.PAUSED;
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                }

                if (!running) {
                    break;
                }
                state = AcceptorState.RUNNING;

                try {
                    //if we have reached max connections, wait
                    countUpOrAwaitConnection();

                    SocketChannel socket = null;
                    try {
                        // Accept the next incoming connection from the server
                        // socket
                        socket = serverSock.accept();
                    } catch (IOException ioe) {
                        //we didn't get a socket
                        countDownConnection();
                        // Introduce delay if necessary
                        errorDelay = handleExceptionWithDelay(errorDelay);
                        // re-throw
                        throw ioe;
                    }
                    // Successful accept, reset the error delay
                    errorDelay = 0;

                    // setSocketOptions() will add channel to the poller
                    // if successful
                    if (running && !paused) {
                        if (!setSocketOptions(socket)) {
                            countDownConnection();
                            closeSocket(socket);
                        }
                    } else {
                        countDownConnection();
                        closeSocket(socket);
                    }
                } catch (SocketTimeoutException sx) {
                    // Ignore: Normal condition
                } catch (IOException x) {
                    if (running) {
                        log.error(sm.getString("endpoint.accept.fail"), x);
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.error(sm.getString("endpoint.accept.fail"), t);
                }
            }
            state = AcceptorState.ENDED;
        }
    }

可以看到就是一个while循环，循环里面不断的accept新的连接。

###2.1.3 countUpOrAwaitConnection

先来看下在accept新的连接之前，首选进行连接数的自增，即countUpOrAwaitConnection

	protected void countUpOrAwaitConnection() throws InterruptedException {
        if (maxConnections==-1) return;
        LimitLatch latch = connectionLimitLatch;
        if (latch!=null) latch.countUpOrAwait();
    }

当我们设置maxConnections=-1的时候就表示不用限制最大连接数。默认是限制10000,如果不限制则一旦出现大的冲击，则tomcat很有可能直接挂掉，导致服务停止。

这里的需求就是当前连接数一旦超过最大连接数maxConnections，就直接阻塞了,一旦当前连接数小于最大连接数maxConnections，就不再阻塞，我们来看下这个功能的具体实现latch.countUpOrAwait()

具体看这个需求无非就是一个共享锁，来看具体实现：

![LimitLatch实现](https://static.oschina.net/uploads/img/201605/01153659_cxxS.png "LimitLatch实现")

目前实现里算是使用了2个锁，LimitLatch本身的AQS实现再加上AtomicLong的AQS实现。也可以不使用AtomicLong来实现。

共享锁的tryAcquireShared实现中，如果不依托AtomicLong，则需要进行for循环加CAS的自增，自增之后没有超过limit这里即maxConnections，则直接返回1表示获取到了共享锁，如果一旦超过limit则首先进行for循环加CAS的自减，然后返回-1表示获取锁失败，便进入加入同步队列进入阻塞状态。

共享锁的tryReleaseShared实现中，该方法可能会被并发执行，所以释放共享锁的时候也是需要for循环加CAS的自减

上述的for循环加CAS的自增、for循环加CAS的自减的实现全部被替换成了AtomicLong的incrementAndGet和decrementAndGet而已。


上文我们关注的latch.countUpOrAwait()方法其实就是在获取一个共享锁，如下：

	/**
     * Acquires a shared latch if one is available or waits for one if no shared
     * latch is current available.
     * @throws InterruptedException If the current thread is interrupted
     */
    public void countUpOrAwait() throws InterruptedException {
        if (log.isDebugEnabled()) {
            log.debug("Counting up["+Thread.currentThread().getName()+"] latch="+getCount());
        }
        sync.acquireSharedInterruptibly(1);
    }

###2.1.4 连接的处理

从上面可以看到在真正获取一个连接之前，首先是把连接计数先自增了。一旦TCP三次握手成功连接建立，就能从ServerSocketChannel的accept方法中获取到新的连接了。一旦获取连接或者处理过程发生异常则需要将当前连接数自减的，否则会造成连接数虚高，即当前连接数并没有那么多，但是当前连接数却很大，一旦超过最大连接数，就导致其他请求全部阻塞，没有办法被ServerSocketChannel的accept处理。该bug在Tomcat7.0.26版本中出现了，详细见这里的一篇文章[Tomcat7.0.26的连接数控制bug的问题排查](http://ifeve.com/tomcat7-0-26-connect-bug/)

然后我们来看下，一个SocketChannel连接被accept获取之后如何来处理的呢？

	protected boolean setSocketOptions(SocketChannel socket) {
        // Process the connection
        try {
            //disable blocking, APR style, we are gonna be polling it
            socket.configureBlocking(false);
            Socket sock = socket.socket();
            socketProperties.setProperties(sock);

            NioChannel channel = nioChannels.pop();
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
                } else {
                    channel = new NioChannel(socket, bufhandler);
                }
            } else {
                channel.setIOChannel(socket);
                channel.reset();
            }
            getPoller0().register(channel);
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error("",t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(t);
            }
            // Tell to close the socket
            return false;
        }
        return true;
    }

处理过程如下：

-	设置非阻塞，以及其他的一些参数如SoTimeout、ReceiveBufferSize、SendBufferSize

-	然后将SocketChannel封装成一个NioChannel，封装过程使用了缓存，即避免了重复创建NioChannel对象，直接利用原有的NioChannel，并将NioChannel中的数据全部清空。也正是这个缓存也造成了一次bug，详见[断网故障时Mtop触发tomcat高并发场景下的BUG排查和修复（已被apache采纳）](https://yq.aliyun.com/articles/2889?spm=5176.team22.teamshow1.30.XRi499#)

-	选择一个Poller进行注册

下面就来详细介绍下Poller

##2.2 Poller

###2.2.1 初始化过程

	// Start poller threads
    pollers = new Poller[getPollerThreadCount()];
    for (int i=0; i<pollers.length; i++) {
        pollers[i] = new Poller();
        Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
        pollerThread.setPriority(threadPriority);
        pollerThread.setDaemon(true);
        pollerThread.start();
    }

前面没有说到Poller的数量控制，来看下

	/**
     * Poller thread count.
     */
    private int pollerThreadCount = Math.min(2,Runtime.getRuntime().availableProcessors());
    public void setPollerThreadCount(int pollerThreadCount) { this.pollerThreadCount = pollerThreadCount; }
    public int getPollerThreadCount() { return pollerThreadCount; }

如果不设置的话最大就是2

###2.2.2 Poller注册SocketChannel

来详细看下getPoller0().register(channel)：

	public Poller getPoller0() {
        int idx = Math.abs(pollerRotater.incrementAndGet()) % pollers.length;
        return pollers[idx];
    }

就是轮训一个Poller来进行SocketChannel的注册

	/**
     * Registers a newly created socket with the poller.
     *
     * @param socket    The newly created socket
     */
    public void register(final NioChannel socket) {
        socket.setPoller(this);
        NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
        socket.setSocketWrapper(ka);
        ka.setPoller(this);
        ka.setReadTimeout(getSocketProperties().getSoTimeout());
        ka.setWriteTimeout(getSocketProperties().getSoTimeout());
        ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
        ka.setSecure(isSSLEnabled());
        ka.setReadTimeout(getSoTimeout());
        ka.setWriteTimeout(getSoTimeout());
        PollerEvent r = eventCache.pop();
        ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
        if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
        else r.reset(socket,ka,OP_REGISTER);
        addEvent(r);
    }

	private void addEvent(PollerEvent event) {
        events.offer(event);
        if ( wakeupCounter.incrementAndGet() == 0 ) selector.wakeup();
    }

	private final SynchronizedQueue<PollerEvent> events =
                new SynchronizedQueue<>();

这里又是进行一些参数包装，将socket和Poller的关系绑定，再次从缓存中取出或者重新构建一个PollerEvent，然后将该event放到Poller的事件队列中等待被异步处理

###2.2.3 Poller的run方法

在Poller的run方法中不断处理上述事件队列中的事件，直接执行PollerEvent的run方法，将SocketChannel注册到自己的Selector上。

	public boolean events() {
        boolean result = false;

        PollerEvent pe = null;
        while ( (pe = events.poll()) != null ) {
            result = true;
            try {
                pe.run();
                pe.reset();
                if (running && !paused) {
                    eventCache.push(pe);
                }
            } catch ( Throwable x ) {
                log.error("",x);
            }
        }

        return result;
    }

并将Selector监听到的IO读写事件封装成SocketProcessor，交给线程池执行

	SocketProcessor sc = processorCache.pop();
    if ( sc == null ) sc = new SocketProcessor(attachment, status);
    else sc.reset(attachment, status);
    Executor executor = getExecutor();
    if (dispatch && executor != null) {
        executor.execute(sc);
    } else {
        sc.run();
    }

我们来看看这个线程池的初始化：

	public void createExecutor() {
        internalExecutor = true;
        TaskQueue taskqueue = new TaskQueue();
        TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
        executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
        taskqueue.setParent( (ThreadPoolExecutor) executor);
    }

就是创建了一个ThreadPoolExecutor，那我们就重点关注下核心线程数、最大线程数、任务队列等信息

	private int minSpareThreads = 10;
    public int getMinSpareThreads() {
        return Math.min(minSpareThreads,getMaxThreads());
    }

核心线程数最大是10个，再来看下最大线程数

	private int maxThreads = 200;

默认就是上面的配置参数maxThreads为200。还有就是TaskQueue，这里的TaskQueue是LinkedBlockingQueue<Runnable>的子类，最大容量就是Integer.MAX_VALUE，根据之前ThreadPoolExecutor的源码分析，核心线程数满了之后，会先将任务放到队列中，队列满了才会创建出新的非核心线程，如果队列是一个大容量的话，也就是不会到创建新的非核心线程那一步了。

但是这里的TaskQueue修改了底层offer的实现

	public boolean offer(Runnable o) {
      //we can't do any checks
        if (parent==null) return super.offer(o);
        //we are maxed out on threads, simply queue the object
        if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);
        //we have idle threads, just add it to the queue
        if (parent.getSubmittedCount()<(parent.getPoolSize())) return super.offer(o);
        //if we have less threads than maximum force creation of a new thread
        if (parent.getPoolSize()<parent.getMaximumPoolSize()) return false;
        //if we reached here, we need to add it to the queue
        return super.offer(o);
    }

这里当线程数小于最大线程数的时候就直接返回false即入队列失败，则迫使ThreadPoolExecutor创建出新的非核心线程。

TaskQueue这一块没太看懂它的意图是什么，有待继续研究。


#3 结束语

本篇文章描述了tomcat8.5中的NIO线程模型，以及其中涉及到的相关参数的设置。下一篇简单整理下tomcat的整体架构图