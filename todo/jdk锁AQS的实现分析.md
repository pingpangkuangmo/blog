# 1 目录

# 2 Lock和Condition接口

最近打算将jdk的Lock系列详细的分析一下，解决以下几个问题：

-	AQS详细分析
-	独占锁、共享锁、读写锁
-	是否可重入
-	公平锁和非公平锁的区别，这里的公平和非公平是针对哪些线程来说的？代码又如何实现？

Lock和synchronized是对应的，Condition和Object的监控方法（wait、notify）是对应的

##2.1 Lock接口

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

