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

AQS提供了上述线程安全和非安全的设置状态state的方法，供我们在实现锁的tryAcquire、tryRelease等方法的时候合理的时候它们。

## 3.3 AbstractQueuedSynchronizer的FIFO队列

就以获取独占锁为例来详细看下该过程：

	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

前面简单提到了，就是先拿锁实现的tryAcquire方法去尝试获取独占锁，一旦获取锁失败就进入队列，交给AQS来处理。AQS的处理简单描述下就是将当前线程包装成Node节点然后放到队列中进程排队，等待前面的Node节点都出队了，被唤醒轮到自己再次去竞争锁。

我们先来认识下Node节点,大致如下结构：

	static final class Node {
      
	    volatile Node prev;
	    volatile Node next;
	
	    volatile Thread thread;
	
	    Node nextWaiter;

		volatile int waitStatus;
	}

	private transient volatile Node head;

	private transient volatile Node tail;

首先就是prev、next节点可以构成一个双向队列。AQS中含有上述的head和tail两个属性一起来构成FIFO队列。

Thread thread则代表的是构成此节点的线程。

Node nextWaiter：是用于condition queue，用于构成一个单向的FIFO队列，详见下面。

volatile int waitStatus：则表示该节点当前的状态，目前有如下状态：

	/** waitStatus value to indicate thread has cancelled */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int PROPAGATE = -3;

CANCELLED：表示该节点所对应的线程因为获取锁的过程中超时或者被中断而被设置成此状态

SIGNAL：表示该节点所对应的线程被阻塞，不再被调度执行，需要等待其他线程释放锁之后来唤醒它，才能再次加入锁的竞争

CONDITION：表示该节点所对应的线程被阻塞，不再被调度执行，在等待某一个condition的signal、signalAll方法的唤醒

PROPAGATE：只用于共享状态的HEAD节点，目前还没弄清楚，欢迎一起来探讨

节点创建后默认状态值是0。

接下来我们就要分别看下这个独占锁的入队和出队过程以及共享锁的入队和出队过程

###3.3.1 独占锁的获取和释放过程

先来看看获取

	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

首先调用锁自己的tryAcquire业务逻辑来尝试获取锁，一旦获取失败，则进入AQS的处理流程，即acquireQueued方法

第一步：就是构造一个Node并入队

	private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

	private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

这里先尝试通过CAS将新添加的Node节点放置当前tail的后面，如果tail为空或者CAS失败，就进入下面的for循环形式的CAS流程，前面的尝试主要是为了针对大部分情况即（pred!=null的情况下）能够快速处理，而for循环则是要考虑所有情况，因为判断逻辑就比较多了。如刚初始化的时候，head和tail都指向了一个空的Node，该Node并不需要获取锁。

第二步：入队后的处理逻辑，是被阻塞呢？还是持续不断尝试获取锁呢？

	final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

这个就要查看新构建的Node的前一个节点是不是head,如果是head，则该新节点可以尝试下获取锁，一旦获取锁成功就会设置head指向当前节点Node。

来重点说说这个head节点：刚初始化的时候head和tail都指向的是一个空的Node，head节点并没有获取到锁，见上面。如果上述尝试获取锁成功就从新设置head节点为当前Node，此时head节点又是一个获取了锁的节点。

如果当前节点的前一个节点不是head或者是head但是尝试获取锁失败，此时就需要衡量下是否需要将当前节点阻塞，即shouldParkAfterFailedAcquire方法的逻辑：

	private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

如果前置节点处于Node.SIGNAL状态，则该Node则需要被阻塞，即还轮不到它。

如果前置节点状态处于Node.CANCELLED状态，则该Node需要重新更换前置节点

如果前置节点状态处于其他状态，则需要把前置节点的状态值设置成Node.SIGNAL，以便下一次循环尝试获取锁失败时，该节点被阻塞，即满足上述第一种情况。从这里看出，节点的状态大部分是应用于后续节点行为判断使用的，而对自身的逻辑并没什么影响。

基本上无论如何，该节点没有机会尝试获取锁或者有机会尝试获取锁但是又失败时，最终都会被阻塞，则里的阻塞使用的就是LockSupport.park：

	private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

接下来看看释放

	public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

	private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }

先调用锁的tryRelease具体业务逻辑，然后就会使用LockSupport.unpark唤醒head的下一个节点，至此在上述acquireQueued方法中被阻塞的那个Node（head的下一个Node）不再阻塞，再次继续for循环流程，又一次开始尝试，如果获取成功，则更新了head节点，则之前的head就表示出队了，如果获取还失败，说明又被外部线程获取到了，那就再一次的被park阻塞。

共享锁的获取和释放过程就不再说了，大部分都是相同的。

## 3.4 AbstractQueuedSynchronizer的condition queue

来看下AQS提供的Condition实现，简单如下

	public class ConditionObject implements Condition{
	    private transient Node firstWaiter;
	    private transient Node lastWaiter;
	}

一个ConditionObject对象本身维护了一个FIFO单向队列，这里通过Node的Node nextWaiter属性来建立关联。

我们来简单看看Condition的await和signal:

	public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }

这里不再一点一旦详细研究了，内容太多了，简单看下整体的逻辑：

前提：在调用Condition的await和signal的方法前必须先获取到锁

第一步：首先构造一个Node，初始化状态为Node.CONDITION，并放到对应ConditionObject的lastWaiter上

第二步：释放锁

第三步：while循环里检查该Node是否在同步队列即上述的FIFO双向队列，不在的话，则被park住，等待unpark的唤醒

第四步：收到了unpark唤醒（在唤醒的时候就顺便将该Node加入了上述的FIFO双向队列中了，因此可以跳出while循环了），跳出了while循环

第五步：该Node已经进入了上述的FIFO双向队列中了，开始了上面介绍的逻辑

再来看看唤醒操作：

	final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

就是unpark头节点Node，并且将该Node放入上述的FIFO双向队列中。

#4 结束语

下一篇就来详细的介绍几个具体的基于AQS的锁的实现。