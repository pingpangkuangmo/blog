# 1 目录

# 2 Lock和Condition接口

最近打算将jdk的Lock系列详细的分析一下，解决以下几个问题：

-	AQS详细分析
-	独占锁、共享锁、读写锁
-	是否可重入
-	公平锁和非公平锁的区别，这里的公平和非公平是针对哪些线程来说的？代码又如何实现？

Lock和synchronized是对应的，Condition和Object的监控方法（wait、notify）是对应的

## 2.1 Lock接口

先来大致看下接口方法

	public interface Lock {

	    void lock();
		void lockInterruptibly() throws InterruptedException;
	    
	    boolean tryLock();
	    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

	    void unlock();

	    Condition newCondition();
	}

lock:在获取锁之前一直阻塞，并且不接受线程中断响应

lockInterruptibly：在获取锁之前一直阻塞，但是接受线程中断响应，即一旦该线程被中断，会退出阻塞，抛出InterruptedException异常，该方法执行结束

tryLock：尝试获取锁，不管成不成功立马返回，返回结果代表是否成功获取了锁

tryLock(long time, TimeUnit unit)：在一段时间内不断尝试获取锁，如果超时还未获取锁则抛出InterruptedException异常，该方法执行结束

unlock：释放锁，要和上面的获取锁方法成对出现

newCondition：使用该锁获取一个Condition，Condition是和生产它的锁是息息相关，绑定在一起的。

##2.2 Condition接口

先来大致看下接口方法

	public interface Condition {

	    void await() throws InterruptedException;
	    void awaitUninterruptibly();
	    boolean await(long time, TimeUnit unit) throws InterruptedException;
	    boolean awaitUntil(Date deadline) throws InterruptedException;
	   
	    void signal();
	    void signalAll();
	}

就await和signal方法，和Object的wait、notify是相对应的。

await：阻塞，一直等待到该对象的signal或者signalAll被调用，退出阻塞，或者该线程被中断，抛出InterruptedException异常，方法执行结束

awaitUninterruptibly：和上面差不多，但是不响应线程的中断，即线程中断对它没影响

await(long time, TimeUnit unit)和awaitUntil(Date deadline)：等待一段时间，一旦超时或者线程被中断，都会抛出InterruptedException异常，方法执行结束

signal：唤醒一个处于await的线程，这里都是针对同一个Condition对象来说的

signalAll：唤醒所有处于await的线程，这里都是针对同一个Condition对象来说的


# 3 AbstractQueuedSynchronizer实现分析


## 3.1 AbstractQueuedSynchronizer方法概述

我们经常会听说AQS，全称就是这个AbstractQueuedSynchronizer。它在锁中扮演什么角色呢？

Lock的接口方法是针对用户的使用而定义的，我们在实现Lock的时候，就需要做如下事情，重点关注下这些事情的共性和异性

-	指明什么情况下才叫获取到锁：如独占锁，一旦有人占据了，就不能获取到锁。如共享锁，有人占据，但是没超过限制也能获取锁。这一部分应该是锁的实现的业务代码，每种锁都有自己的业务逻辑。这一部分其实就是AbstractQueuedSynchronizer对子类留出的tryAcquire方法

-	获取不到锁的时候该如何处理呢：我们当然希望它们能够继续等待，有一个队列就最好不过了，一旦获取锁失败就加入到等待队列中排队，队列中的等待者依次再去竞争获取锁。这一部分代码其实就和哪种锁没太大关系了，所以应该是锁的共性部分，这一部分其实就是AbstractQueuedSynchronizer实现的共性部分acquireQueued

那么针对Lock接口定义的lock方法实现逻辑就一般如下了：

	if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))

代表着先试着获取一下锁，如果获取不成功，就把之后的处理逻辑交给AbstractQueuedSynchronizer来处理。反正你只管告诉AbstractQueuedSynchronizer它怎样才叫获取到锁，其他的等待处理逻辑你就可以不用再关心了。

以上也仅仅是部分内容，下面来全面看下AbstractQueuedSynchronizer对外留出的接口和已实现的共性部分

对外留出的接口：

-	tryAcquire：该方法向AQS解释了怎么才叫获取一把独占锁

-	tryRelease：该方法向AQS解释了怎么才叫释放一把独占锁

-	tryAcquireShared：该方法向AQS解释了怎么才叫获取一把共享锁

-	tryReleaseShared：该方法向AQS解释了怎么才叫释放一把共享锁

这些内容就是各种锁本身的业务逻辑，属于异性部分。

来看看锁的共性部分相关方法：

-	acquire：获取一把独占锁的过程

		public final void acquire(int arg) {
	        if (!tryAcquire(arg) &&
	            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
	            selfInterrupt();
	    }
	
	就是先拿锁实现的tryAcquire方法去尝试获取独占锁，一旦获取锁失败就进入队列，交给AQS来处理，一旦成功就表示获取到了一把锁。


-	release：释放一把独占锁的过程

		public final boolean release(int arg) {
	        if (tryRelease(arg)) {
	            Node h = head;
	            if (h != null && h.waitStatus != 0)
	                unparkSuccessor(h);
	            return true;
	        }
	        return false;
	    }

	就是先拿锁实现的tryRelease方法尝试释放独占锁，一旦释放成功，就通知队列，有人释放锁了，队列前面的可以再次去竞争锁了（这一部分下面详细说明）

-	acquireShared：获取一把共享锁的过程

		public final void acquireShared(int arg) {
	        if (tryAcquireShared(arg) < 0)
	            doAcquireShared(arg);
	    }

	就是先拿锁实现的tryAcquireShared方法尝试获取共享锁，一旦获取失败，就进入队列，交给AQS来处理

-	releaseShared：释放一把共享锁的过程

		public final boolean releaseShared(int arg) {
	        if (tryReleaseShared(arg)) {
	            doReleaseShared();
	            return true;
	        }
	        return false;
	    }
	
	就是先拿锁实现的tryReleaseShared方法尝试释放共享锁，一旦释放成功，就通知队列，有人释放锁了

一个Lock只要实现了AQS的留出的业务部分代码，就可以使用AQS提供的上述方法来实现锁的相关功能，如一个简单的独占锁实现如下（略去其他一些代码）：

	public class Mutex implements Lock{

		@Override
		public void lock() {
			sync.acquire(1);
		}
	
		@Override
		public void unlock() {
			sync.release(1);
		}
		
		private final Sync sync=new Sync();
		
		private static class Sync extends AbstractQueuedSynchronizer{
	
			@Override
			protected boolean tryAcquire(int arg) {
				if(compareAndSetState(0,1)){
					setExclusiveOwnerThread(Thread.currentThread());
					return true;
				}
				return false;
			}
	
			@Override
			protected boolean tryRelease(int arg) {
				if(getState()==0){
					throw new IllegalMonitorStateException();
				}
				setExclusiveOwnerThread(null);
				setState(0);
				return true;
			}	
		}
	
	}

## 3.2 AbstractQueuedSynchronizer的状态属性state

从上面我们就看到老是方法中会有各种的int参数，其实这是AbstractQueuedSynchronizer将获取锁的过程量化对数字的操作，而state变量就是用于记录当前数字值的。以独占锁为例:

-	state=0 表示该锁未被其他线程获取

-	一旦有线程想获取锁，就可以对state进行CAS增量操作，这个增量可以是任意值，不过大多数都默认取1。也就是说一旦一个线程对state CAS操作成功就代表该线程获取到了锁，则state就变成1了。其他操作对state CAS操作失败的就代表没获取到锁，就自动进入AQS管理流程

-	其他线程发现当前state值不等于0表示锁已被其他线程获取，就自动进入AQS管理流程

-	一旦获取锁的线程想释放锁，就可以对state进行自减，即减到0，其他线程又可以去获取锁了

从上面的例子中可以看到对state的几种线程安全和非安全操作：

-	compareAndSetState(int expect, int update)：线程安全的设置状态，因为可能多个线程并发调用该方法，所以需要CAS来保证线程安全

-	setState(int newState)：非线程安全的设置状态，这种一般是针对只有一个线程获取锁的时候来释放锁，此时没有并发的可能性，所以就不需要上述的compareAndSetState操作

-	getState()：获取状态值

## 3.3 AbstractQueuedSynchronizer的FIFO队列

就以获取独占锁为例来详细看下该过程：

	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

前面简单提到了，就是先拿锁实现的tryAcquire方法去尝试获取独占锁，一旦获取锁失败就进入队列，交给AQS来处理。AQS的处理简单描述下就是将当前线程包装成Node节点然后放到队列中进程排队，等待前面的Node节点都出队了，被唤醒轮到自己再次去竞争锁。



## 3.4 AbstractQueuedSynchronizer的condition queue

