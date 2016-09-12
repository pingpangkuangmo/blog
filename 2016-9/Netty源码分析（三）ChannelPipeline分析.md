准备将Netty的源码过一下，一来对自己是个总结消化的过程，二来希望对那些打算看Netty源码的人（已经熟悉Netty的Reactor模型）能有一些帮助。目前所看Netty版本是4.1.3.Final。

# 1 目录

-	[Netty概览](http://my.oschina.net/pingpangkuangmo/blog/734051)
-	[EventLoopGroup分析](http://my.oschina.net/pingpangkuangmo/blog/742929)

# 2 EventLoopGroup和EventLoop介绍

## 2.1 EventLoopGroup介绍

在前面一篇文章中提到了，EventLoopGroup主要负责2个事情，这里再重复下：

它主要包含2个方面的功能，注册Channel和执行一些Runnable任务。

![EventLoopGroup介绍](https://static.oschina.net/uploads/img/201608/06184758_9ZPc.png "EventLoopGroup介绍")

**功能1：先来看看注册Channel**，即将Channel注册到Selector上，由Selector来调度Channel的相关事件，如读、写、Accept等事件。

而EventLoopGroup的设计是，它包含多个EventLoop（每一个EventLoop通常内部包含一个线程），在执行上述注册过程中是需要选择其中的一个EventLoop来执行上述注册行为，这里就出现了一个选择策略的问题，该选择策略接口是EventExecutorChooser，你也可以自定义一个实现。

从上面可以看到，EventLoopGroup做的工作大部分是一些总体性的工作如初始化上述多个EventLoop、EventExecutorChooser等，具体的注册Channel还是交给它内部的EventLoop来实现。

**功能2：执行一些Runnable任务**

EventLoopGroup继承了EventExecutorGroup，EventExecutorGroup也是EventExecutor的集合，EventExecutorGroup也是掌管着EventExecutor的初始化工作，EventExecutorGroup对于Runnable任务的执行也是选择内部中的一个EventExecutor来做具体的执行工作。

netty中很多任务都是异步执行的，一旦当前线程要对某个EventLoop执行相关操作，如注册Channel到某个EventLoop，如果当前线程和所要操作的EventLoop内部的线程不是同一个，则当前线程就仅仅向EventLoop提交一个注册任务，对外返回一个ChannelFuture。

总结：EventLoopGroup含有上述2种功能，它更多的是一个集合，但是具体的功能实现还是选择内部的一个item元素来执行相关任务。
这里的内部item元素通常即实现了EventLoop，又实现了EventExecutor，如NioEventLoop等

继续来看看EventLoopGroup的整体类图

![输入图片说明](https://static.oschina.net/uploads/img/201609/05145206_bki3.png "在这里输入图片标题")

从图中可以看到有2路分支：

1 MultithreadEventLoopGroup：用于封装多线程的初始化逻辑，指定线程数等，即初始化对应数量的EventLoop，每个EventLoop分配到一个线程

![MultithreadEventLoopGroup的初始化](https://static.oschina.net/uploads/img/201609/05150646_GNJr.png "MultithreadEventLoopGroup的初始化")

上图中的newChild方法，NioEventLoopGroup就采用NioEventLoop作为实现，EpollEventLoopGroup就采用EpollEventLoop作为实现

如NioEventLoopGroup的实现：

	protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
    }

2 EventLoop接口实现了EventLoopGroup接口，主要因为EventLoopGroup中的功能接口还是要靠内部的EventLoop来完成具体的操作


## 2.2 EventLoop介绍

EventLoop主要工作就是注册Channel，并负责监控管理Channel的读写等事件，这就涉及到不同的监控方式，linux下有3种方式来进行事件监听

select、poll、epoll

目前java的Selector接口的实现如下：

-	PollSelectorImpl：实现了poll方式
-	EPollSelectorImpl：实现了epoll方式

而Netty呢则使用如下：

-	NioEventLoop：采用的是jdk Selector接口（使用PollSelectorImpl的poll方式）来实现对Channel的事件检测

-	EpollEventLoop：没有采用jdk Selector的接口实现EPollSelectorImpl，而是Netty自己实现的epoll方式来实现对Channel的事件检测，所以在EpollEventLoop中就不存在jdk的Selector。


### 2.2.1 NioEventLoop介绍

对于NioEventLoopGroup的功能，NioEventLoop都要做实际的实现，NioEventLoop既要实现注册功能，又要实现运行Runnable任务

对于注册Channel：NioEventLoop将Channel注册到NioEventLoop内部的PollSelectorImpl上，来监听该Channel的读写事件

对于运行Runnable任务：NioEventLoop的父类的父类SingleThreadEventExecutor实现了运行Runnable任务，在SingleThreadEventExecutor中，有一个任务队列还有一个分配的线程

	private final Queue<Runnable> taskQueue;
	private volatile Thread thread;

NioEventLoop在该线程中不仅要执行Selector带来的IO事件，还要不断的从上述taskQueue中取出任务来执行这些非IO事件。下面我们来详细看下这个过程

	protected void run() {
	    for (;;) {
	        try {
	            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
	                case SelectStrategy.CONTINUE:
	                    continue;
	                case SelectStrategy.SELECT:
	                    select(wakenUp.getAndSet(false));
	                    if (wakenUp.get()) {
	                        selector.wakeup();
	                    }
	                default:
	                    // fallthrough
	            }
	            cancelledKeys = 0;
	            needsToSelectAgain = false;
	            final int ioRatio = this.ioRatio;
	            if (ioRatio == 100) {
	                processSelectedKeys();
	                runAllTasks();
	            } else {
	                final long ioStartTime = System.nanoTime();
	
	                processSelectedKeys();
	
	                final long ioTime = System.nanoTime() - ioStartTime;
	                runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
	            }
	
	            if (isShuttingDown()) {
	                closeAll();
	                if (confirmShutdown()) {
	                    break;
	                }
	            }
	        } catch (Throwable t) {
	            ...
	        }
	    }
	}

来详细说下这个过程：

-	1 计算当前是否需要执行select过程

	如果当前没有Runnable任务，则执行select（这个select过程稍后详细来说）。
	
	如果当前有Runnable任务，则要去执行处理流程，此时顺便执行下selector.selectNow()，万一有事件发生那就赚了，没有白走这次处理流程

-	2 根据IO任务的时间占比设置来执行IO任务和非IO任务，即上面提到的Runnable任务

	如果ioRatio=100则每次都是执行全部的IO任务，执行全部的非IO任务
	默认ioRatio=50，即一半时间用于处理IO任务，另一半时间用于处理非IO任务。怎么去控制非IO任务所占用时间呢？

	这里是每执行64个非IO任务（这里可能是每个非IO任务比较短暂，减少一些判断带来的消耗）就判断下占用时间是否超过了上述时间限制

接下来详细看下上述select过程

	Selector selector = this.selector;
	try {
	    int selectCnt = 0;
	    long currentTimeNanos = System.nanoTime();
	    long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
	    for (;;) {
	        long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
	        if (timeoutMillis <= 0) {
	            if (selectCnt == 0) {
	                selector.selectNow();
	                selectCnt = 1;
	            }
	            break;
	        }
	
	        // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
	        // Selector#wakeup. So we need to check task queue again before executing select operation.
	        // If we don't, the task might be pended until select operation was timed out.
	        // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
	        if (hasTasks() && wakenUp.compareAndSet(false, true)) {
	            selector.selectNow();
	            selectCnt = 1;
	            break;
	        }
	
	        int selectedKeys = selector.select(timeoutMillis);
	        selectCnt ++;
	
	        if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
	            // - Selected something,
	            // - waken up by user, or
	            // - the task queue has a pending task.
	            // - a scheduled task is ready for processing
	            break;
	        }
	        if (Thread.interrupted()) {
	            // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
	            // As this is most likely a bug in the handler of the user or it's client library we will
	            // also log it.
	            //
	            // See https://github.com/netty/netty/issues/2426
	            if (logger.isDebugEnabled()) {
	                logger.debug("Selector.select() returned prematurely because " +
	                        "Thread.currentThread().interrupt() was called. Use " +
	                        "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
	            }
	            selectCnt = 1;
	            break;
	        }
	
	        long time = System.nanoTime();
	        if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
	            // timeoutMillis elapsed without anything selected.
	            selectCnt = 1;
	        } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
	                selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
	            // The selector returned prematurely many times in a row.
	            // Rebuild the selector to work around the problem.
	            logger.warn(
	                    "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
	                    selectCnt, selector);
	
	            rebuildSelector();
	            selector = this.selector;
	
	            // Select again to populate selectedKeys.
	            selector.selectNow();
	            selectCnt = 1;
	            break;
	        }
	        currentTimeNanos = time;
	    }
	} catch (CancelledKeyException e) {
		...
	}

-	1 首先计算此次select过程的截止时间

		protected long delayNanos(long currentTimeNanos) {
	        ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
	        if (scheduledTask == null) {
	            return SCHEDULE_PURGE_INTERVAL;
	        }
	
	        return scheduledTask.delayNanos(currentTimeNanos);
	    }

	这里其实就是从一个定时 任务队列中取出定时任务，如果有则计算出离当前定时任务的下一次执行时间之差，如果没有则按照固定的1s作为select过程的时间

-	2 将当前时间差转化成ms

	如果当前时间差不足0.5ms的话，即timeoutMillis<=0，并且是第一次执行，则认为时间太短执行执行一次selectNow

	
-	3 如果有任务，则立即执行一次selectNow，跳出for循环

-	4 然后就是普通的selector.select(timeoutMillis)

	在这段时间内如果有事件则跳出for循环，如果没有事件则已经花费对应的时间差了，再次执行for循环，计算的timeoutMillis就会小于0，也会跳出for循环

	在上述逻辑中，基本selectCnt都是1，不会出现很多次，而这里针对selectCnt有很多次的处理是基于一个情况：

		selector.select(timeoutMillis)

	Selector的正常逻辑是一旦有事件就返回，没有事件则最多等待timeoutMillis时间。
	然而底层操作系统实现可能有bug，会出现：即使没有产生事件就直接返回了，并没有按照要求等待timeoutMillis时间。
	
	现在的解决办法就是：
	记录上述出现的次数，一旦超过512这个阈值（可设置），就重新建立新的Selector，并将之前的Channel也全部迁移到新的Selector上

至此，NioEventLoop的主逻辑流程就介绍完了，之后就该重点介绍其中对于IO事件的处理了。然后就会引出来ChannelPipeline的处理流程

### 2.2.2 EpollEventLoop介绍

EpollEventLoop和NioEventLoop的主流程逻辑基本上是差不多的，不同之处就在于EpollEventLoop用epoll方式替换NioEventLoop中的PollSelectorImpl的poll方式。

这里不再详细说明了，之后会详细的说明Netty的epoll方式和jdk中的epoll方式的区别。

# 3 后续

下一篇就要详细描述下NioEventLoop对于IO事件的处理，即ChannelPipeline的处理流程。
