#1 系列目录

-	[线程池接口分析以及FutureTask设计实现](http://my.oschina.net/pingpangkuangmo/blog/666762)
-	[ThreadPoolExecutor实现分析]()

该系列打算从一个最简单的Executor执行器开始一步一步扩展到ThreadPoolExecutor，希望能粗略的描述出线程池的各个实现细节。针对JDK1.7中的线程池

#2 ThreadPoolExecutor

从上一篇文章中了解到：核心execute(futureTask)方法需要被子类来实现，所以我们就俩重点看看ThreadPoolExecutor是如何实现这个核心方法的

##2.1 ThreadPoolExecutor的参数

	public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }

-	int corePoolSize：核心线程数

-	int maximumPoolSize：最大线程数

-	BlockingQueue workQueue：任务队列

-	long keepAliveTime：和TimeUnit unit一起构成线程的最大空闲时间，一旦超过该时间还没有任务处理，该线程就走向结束了。它是针对当前线程数已经超过corePoolSize核心线程数了或者核心线程数也开启超时策略，即属性allowCoreThreadTimeOut=true

-	ThreadFactory threadFactory：线程工厂

-	RejectedExecutionHandler handler：拒绝策略，当任务太多来不及处理，可拒绝该任务

先简单描述下ThreadPoolExecutor的execute(futureTask)过程的大概情况：

1	如果当前线程数小于corePoolSize，则直接创建出一个线程，用于执行新加进来的任务

2	如果当前线程数已经超过corePoolSize，则将该任务放到BlockingQueue workQueue任务队列中，该任务队列可以是有限容量也可以是无限容量的。每个线程处理完一个任务后，都会不断的从BlockingQueue workQueue任务队列中取出任务并执行

3	如果BlockingQueue workQueue是有限容量的，已满无法放进新的任务了，如果此时的线程数小于maximumPoolSize，则直接创建一个线程执行该任务

4	如果线程数已达到maximumPoolSize不能再创建线程了，则直接使用RejectedExecutionHandler handler拒绝该任务

##2.2 execute(futureTask)代码分析



