# 1 目录

-	[Timer和ScheduledThreadPoolExecutor的定时任务](http://my.oschina.net/pingpangkuangmo/blog/745704)

# 2 调度概述

-	1 说到调度，有最简单的Timer、ScheduledThreadPoolExecutor，又有Spring Task、quartz。

-	2 说到分布式调度，有基于数据库实现的quartz集群方案、当当网开源的基于ZooKeeper的elastic-job

-	3 说到大数据方向的调度，如apache的oozie、阿里的zeus。他们更多是定制大数据方向的job，如MapReduce job，spark job，hive job等，还有解决了上面都没触碰的job依赖管理。

-	4 说到集群资源的调度，如Yarn、Mesos。他们则是更高度的抽象，他们把调度拆分成**资源调度**和**任务调度**，他们只负责资源方面的调度。

	**资源调度**：仅仅负责对集群所有机器的CPU 、内存、网络等资源进行统一规划和管理，有任务到来时，通过合理的分配（达到资源充分利用的目的）挑选出对应资源交付给任务来执行。

	**任务调度**：就要提到任务模型了，如MapReduce是一种任务模型，每一种任务模型有有对应的ApplicationMaster来负责任务调度，如MapReduce的就是MRAppMaster。MRAppMaster负责任务的分片，任务的failover，任务之间的逻辑、向Yarn申请资源来执行Map任务或者Reduce任务等等。目前Yarn已经集成了对MapReduce、Spark等任务模型的处理，如果你还想在Yarn上运行自己的任务模型，就需要实现一个自己的ApplicationMaster来负责任务调度。

# 3 Timer

Timer是单线程的，比较简单，一个线程TimerThread 加一个任务队列TaskQueue，每一个任务都是TimerTask类型的。

提供了如下方式来调度任务的执行，这个就不再说了，自己看下文档

![Timer的调度方法](https://static.oschina.net/uploads/img/201609/12104540_cvbE.png "Timer的调度方法")

## 3.1 TimerThread执行过程

见下图
![TimerThread的执行过程](https://static.oschina.net/uploads/img/201609/12105744_4rwU.png "TimerThread的执行过程")

-	1 一旦队列是空的，就进行等待，直到队列中有数据
-	2 一旦Timer被取消，就会设置newTasksMayBeScheduled=false，并且清空队列。正常情况下走到这里队列是有数据的，没有数据则说明Timer被取消了，所以这里就退出while循环了，TimerThread执行结束了
-	3 如果queue有任务，则取出最近要执行的任务，查看该任务是否取消了，一旦取消了，就从queue中移除，继续下一次循环
-	4 如果该任务没有取消，则判断是否到达该任务的执行时间了，如果到达即taskFired=true，如果该任务不是周期任务，则直接从queue中删除该任务
-	5 如果该任务时周期性任务，计算出下次执行时间，再将该任务放到queue中（其实是从新调整该任务在队列中的位置）
-	6 如果该任务还没到触发时间，则等待一段时间
-	7 如果该任务触发了，就直接调用任务的run方法

存在的比较严重的问题：这里可以看到，这里并没有对任务的run方法可能抛出的异常进行捕获，就会导致，一旦某个TimerTask任务抛出异常，就会导致TimerThread结束，Timer不可用。所以在使用Timer的时候，自己实现的TimerTask要对可能的异常进行捕获和处理。

## 3.2 TaskQueue对TimerTask的存储

从上面看到，是需要能够从TaskQueue中获取最近要执行的一个任务。如果对所有任务的执行时间进行排序存储也可以实现，但是该场景就没有必要，只需要知道一个最小的执行时间，所以使用二叉堆来进行存储，又分最大堆和最小堆，这里使用最小堆，这里不再详细说明。

最小堆：就是所有的父节点的值都比左右孩子节点的值小。所以最小的值一定是在堆顶。

二叉堆一般使用数组来实现，TaskQueue实现如下：

	private TimerTask[] queue = new TimerTask[128];
	private int size = 0;

size用于标记该数组中任务的个数

添加一个任务的过程：

	void add(TimerTask task) {
        // Grow backing store if necessary
        if (size + 1 == queue.length)
            queue = Arrays.copyOf(queue, 2*queue.length);

        queue[++size] = task;
        fixUp(size);
    }

这里会先判断是否需要扩容，然后会将任务放到数组的末尾，然后调用fixUp（size）方法来调整该任务在数组中的位置，以达到二叉堆的特性。

	private void fixUp(int k) {
        while (k > 1) {
            int j = k >> 1;
            if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }

这里也很简单，就是不断的找出该任务的父节点，判断该任务节点的下一次执行时间和父节点的下一次执行时间的大小，如果小于父节点的话，则与父节点交换位置，继续往上查找父节点再重复上述比较。

# 4 ScheduledThreadPoolExecutor

## 4.1 继承ThreadPoolExecutor

Timer是单线程的，而ScheduledThreadPoolExecutor是多线程的。ScheduledThreadPoolExecutor继承了线程池ThreadPoolExecutor，整体的执行流程不变，还是有那几个核心东西

-	int corePoolSize
-	int maximumPoolSize
-	long keepAliveTime
-	BlockingQueue<Runnable> workQueue
-	ThreadFactory threadFactory
-	RejectedExecutionHandler handler

ThreadPoolExecutor这一部分可以参考我之前的文章[线程池系列分析](http://my.oschina.net/pingpangkuangmo/blog?catalog=3543082)

从ThreadPoolExecutor的原理中可以看到并没有对job的定时调度功能，它里面的Worker并不会去延迟执行一个任务，因为它是通用的，而Timer中的TimerThread是专用的，可以将延迟逻辑放到TimerThread中，而让TaskQueue更轻量的专做简单的二叉堆存储操作，ScheduledThreadPoolExecutor如何来基于ThreadPoolExecutor来实现定时任务呢？

那就只能将TimerThread中做的延迟逻辑放到queue中来做，ScheduledThreadPoolExecutor为此实现了一个DelayedWorkQueue。在取任务的时候会做延迟逻辑。

## 4.2 实现定时功能

我们知道ThreadPoolExecutor会调用BlockingQueue<Runnable> workQueue的offer方法添加任务，会调用task、poll方法来获取任务，来看看DelayedWorkQueue是如何实现这2个操作的

对任务的添加：也是采用二叉堆形式来存放这些任务的，和上述Timer的添加任务方法类似，存储结构如下

	private static final int INITIAL_CAPACITY = 16;
    private RunnableScheduledFuture[] queue = new RunnableScheduledFuture[INITIAL_CAPACITY];
    private int size = 0;

对于任务的获取：普通BlockingQueue<Runnable> workQueue中有数据了，poll方法就可以取到数据，而DelayedWorkQueue则不是，只有当有数据，并且达到任务执行时间了，才能poll出数据，从而实现对定时任务的调度。下面来详细看下这个poll过程

![DelayedWorkQueue的poll过程](https://static.oschina.net/uploads/img/201609/12122100_04M6.png "DelayedWorkQueue的poll过程")

-	1 由于在多线程环境下采用数据来存储，需要使用锁来控制
-	2 然后判断数组中第一个是否为null，如果为null，并且没有等待时间已经用完，就直接返回了，本次poll就没有拿到数据
-	3 如果数组中第一个是null，则等待一段时间，知道时间超时或者被唤醒（在添加任务的时候会进行唤醒操作）
-	4 如果数组中第一个任务不为null，并且到了触发时间，则直接从数组中取出该任务，然后进行一定的调整，是数组中的数据仍然满足二叉堆的性质，然后将取出的任务返回，用于去执行
-	5 如果数组中的任务没有到触发时间，则等待到它的触发时间

# 5 未完待续

下一篇就来说说quartz的简单模式，Spring Task对定时任务的封装，以及quartz基于数据库的集群模式。