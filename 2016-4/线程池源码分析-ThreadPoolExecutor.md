#1 系列目录

-	[线程池接口分析以及FutureTask设计实现](http://my.oschina.net/pingpangkuangmo/blog/666762)
-	[线程池源码分析-ThreadPoolExecutor](http://my.oschina.net/pingpangkuangmo/blog/668520)

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

详细代码如下：

	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

我们来详细看看每一个步骤：

###2.2.1 ctl解析

ctl是ThreadPoolExecutor的一个重要属性，它记录着ThreadPoolExecutor的线程数量和线程状态。

private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

Integer有32位，其中前三位用于记录线程状态，后29位用于记录线程的数量。那线程状态有哪些呢？

	//线程数量占用的位数
	private static final int COUNT_BITS = Integer.SIZE - 3;

	//最大线程数
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl

	//状态值就是只关心前三位的值，所以把后29位清0
    private static int runStateOf(int c)     { return c & ~CAPACITY; }

	//线程数量就是只关心后29位的值，所以把前3位清0
    private static int workerCountOf(int c)  { return c & CAPACITY; }

	//两个数相或
    private static int ctlOf(int rs, int wc) { return rs | wc; }


状态有 RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED

状态的详细信息可以看下源码中的文档说明，这里不再详细说明

###2.2.2 addWorker解析

我们从上面的execute代码中可以看到，当提交一个任务时，当前线程数小于corePoolSize核心线程数的时候，就新添加一个线程，即addWorker(command, true)

我们来详细看下上述过程：

	private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock;
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

addWorker有2种情况，一种就是线程数量不足核心线程数，另一种就是核心线程数已满同时任务队列已满但是线程数不足最大线程数。上述boolean core就是用来区分上述2种情况的。

addWorker首先就要对线程数量自增，即ctl的后29为进行自增。这里就涉及到多线程问题，为了解决多线程问题就采用了for循环加CAS来解决，为什么没有直接用AtomicInteger的incrementAndGet？

incrementAndGet也是内部for循环加CAS，它是要确保一定要自增成功的，而这里我们不一定要自增成功，还要判断当前线程的数量合不合法。
即如果core=true,则当前线程数量不能超过incrementAndGet，如果core=false，则当前线程数量不能超过maximumPoolSize。

如果当前线程数自增成功，下面就需要创建出Worker线程，存放到workers中，如下所述

	private final HashSet<Worker> workers = new HashSet<Worker>();

我们知道HashSet的内部实现就是通过HashMap来实现的，HashMap是线程不安全的，所以在对workers操作的时候必须要要进行加锁，这就用到了ThreadPoolExecutor的mainLock了

	private final ReentrantLock mainLock = new ReentrantLock();

一旦worker新增成功就直接启动该Worker内部的线程。一旦worker新增失败则调用addWorkerFailed处理失败逻辑，什么情况下会失败呢？

-	创建出Worker对象时，内部分配的线程Thread为空，这个线程的创造是由线程工厂ThreadFactory负责的创造的，可能为null

-	在获取mainLock之后，发现当前线程池已被标记成大于SHUTDOWN状态、或者是SHUTDOWN但是firstTask不为null。当线程状态大于SHUTDOWN，当然addWorker要失败。后者怎么解释呢？这里就要详细解释下SHUTDOWN状态，如下所示

		RUNNING:  Accept new tasks and process queued tasks
		SHUTDOWN: Don't accept new tasks, but process queued tasks
		STOP:     Don't accept new tasks, don't process queued tasks,
			      and interrupt in-progress tasks
		TIDYING:  All tasks have terminated, workerCount is zero,
		         the thread transitioning to state TIDYING
		       	will run the terminated() hook method
		TERMINATED: terminated() has completed

SHUTDOWN即不再接收新的task，但是可以继续处理队列中的task。当线程数小于核心线程数的时候，提交的task作为新创建的Worker的firstTask，即firstTask不为null。当线程数大于核心线程数后，此时addWorker中创建的Worker的firstTask就是null,它只负责从队列中取出任务。所以firstTask不为null的时候就表明是新提交的任务，SHUTDOWN状态下是不允许新提交任务的，所以这种情况也要失败

失败之后呢？来详细看失败的处理逻辑，即在addWorkerFailed

	/**
     * Rolls back the worker thread creation.
     * - removes worker from workers, if present
     * - decrements worker count
     * - rechecks for termination, in case the existence of this
     *   worker was holding up termination
     */
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }

-	首先还是先获取mainLock，然后才能对workers操作

-	从workers中删除该Worker，记录的worker数量自减

-	tryTerminate这个有兴趣的可以详细研究

###2.2.2 Worker解析

上面我们看到了仅仅是新增一个Worker，并放到workers中，并启动它，这里来详细看看Worker是如何工作的

![Worker概述](https://static.oschina.net/uploads/img/201605/01003328_FvAP.png "Worker概述")

-	Worker是ThreadPoolExecutor的内部类
-	Worker继承了AQS即AbstractQueuedSynchronizer，用于实现锁的机制，这里先抛出一个问题：为什么要继承AQS
-	实现了Runnable接口，则该Worker就可以作为Thread的参数创建出Thread，线程的运行即运行该Worker的run方法，该方法中会不断的从队列中取出任务并执行

我们来详细看下Worker的run过程

	final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

该方法就是一个while循环，如果有firstTask就运行firstTask的内容，如果没有firstTask就从队列中取出task运行即getTask逻辑。每一个task的运行，都有相应的周期方法调用如

	beforeExecute(wt, task);
	task.run();
	afterExecute(task, thrown);

Worker整个运行过程就是从队列中取出task依次运行上述方法。这里有3个重要地方要关注：

-	getTask逻辑，我们从上面看到while循环中一旦getTask为null就直接退出while循环了，即Worker走向结束了，所以空闲的时候会阻塞在getTask中，一直等到获取到task或者超时。
-	为什么每次执行task都要获取锁
-	worker的退出，就不再详细说明了，各位可自行研究

我们一一来详细说明。

getTask逻辑

	private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            boolean timed;      // Are workers subject to culling?

            for (;;) {
                int wc = workerCountOf(c);
                timed = allowCoreThreadTimeOut || wc > corePoolSize;

                if (wc <= maximumPoolSize && ! (timedOut && timed))
                    break;
                if (compareAndDecrementWorkerCount(c))
                    return null;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

从workQueue队列中获取task有2种方法，一种就是阻塞式获取直到有task任务，另一种就是阻塞一定时间，超时则就直接返回null了。此时返回null意味着Worker就要走向结束了。

当allowCoreThreadTimeOut=true即核心线程也开始timeout计时，或者wc > corePoolSize即当前线程数超过了核心线程数也要开启计时，获取task就采用阻塞一定时间获取，一旦超时即该Worker在keepAliveTime时间内都没获取到task即处于空闲状态，这时候就返回null，即意味着该Worker就走向结束了

其他情况就是不用进行线程空闲计时，即可以一直阻塞直到有task来。

接下来一个重点问题就是每次执行task的时候为什么要先获取锁？

首先该Worker的run方法只可能被一个线程来运行，即该Worker的run方法不可能出现多线程同时运行的情况。那就是Worker有一些资源是多个线程共享的，是什么呢？我们先来看看Worker继承AQS即AbstractQueuedSynchronizer的实现情况

	protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

这里就是一个简单的独占锁，但是重点是不可重入的，重入即当前线程获取锁了，还可以再次获取锁，来简单对比下重入锁的实现

	protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

也就是说Worker本身就是一个简单的独占锁，并且是不可重入的。这个锁的引入到底是为了什么呢？为什么需要在每个task执行前都要获取这个锁呢？这个当时也没太理解，但是要想找出这个原因，可以有如下思路来找：

就看看哪些地方在调用Worker获取锁的方法，获取锁的方法有lock、tryLock

最终发现interruptIdleWorkers会调用tryLock方法，如下：

	private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

该方法就是用于中断那些空闲的Worker，怎么判断一个Worker是否空闲呢？这里就是使用w.tryLock()是否能获取锁来表示一个Worker是否空闲。当Worker在处理任务的时候即不空闲都会获取lock，所以这里就是依据Worker的锁是否被占用了来判定一个Worker是否空闲。

如当重新设置一个ThreadPoolExecutor的核心线程数的时候，如果当前线程数大于了新设置的核心线程数，就需要中断那些空闲的线程

	public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        else if (delta > 0) {
            // We don't really know how many new threads are "needed".
            // As a heuristic, prestart enough new workers (up to new
            // core size) to handle the current number of tasks in
            // queue, but stop if queue becomes empty while doing so.
            int k = Math.min(delta, workQueue.size());
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }

此时再想一想，为什么必须是非重入锁呢？首先说说这里的重入场景：Worker内部的线程不会有重入的问题，始终在执行run方法，Worker本身是一个锁，也就是其他外部线程而非Worker内部的线程在使用Worker的锁的时候是不允许重入的。但是这个不允许重入还有待继续讨论清楚。


#3 结束语

本文重点介绍了ThreadPoolExecutor的几个参数的概念，以及重点分析了execute一个task的过程，还详细介绍了Worker的实现细节。还是有很多代码值得仔细回味和推敲的。下一篇就开始介绍Executors中常见的几种线程池形式。