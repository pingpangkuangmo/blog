# 1系列内容

-	[jdk Selector设计情况](http://my.oschina.net/pingpangkuangmo/blog/733626)
-	[jdk nio poll在linux平台下的实现](http://my.oschina.net/pingpangkuangmo/blog/738216)
-	jdk nio epoll在linux平台下的实现
-	netty 原生epoll在linux平台下的实现
-	epoll的2种通知模式边缘触发、水平触发

# 2 linux的poll实现

linux中有系统调用poll方法，定义如下：

	int poll (struct pollfd *fds, unsigned int nfds, int timeout);

	
上述pollfd结构体定义如下：

	typedef struct pollfd {
		int fd;
		short events;
		short revents;
	} pollfd_t;

int fd：一个文件描述句柄，代表一个Channel连接

short events:该文件描述符感兴趣的事件，如POLLIN表示该文件描述符有读事件，POLLOUT表示该文件描述符可写。

short revents：代表该文件描述符当前已有的事件，如有读事件则值为POLLIN，有读写事件则为POLLIN和POLLOUT的并集

整体的意思就是：你指定了结构体列表的起始地址和要监控的结构体个数，linux系统就会为你在timeout时间内监控上述结构体列表中的文件描述符的相关事件，并把发生的事件写入到上述的short revents属性中。

所以我们在执行一次poll之后，要想获取所有的发生了事件的文件描述符，则需要遍历整个pollfd列表，依次判断上述的short revents是否不等于0，不等于0代表发生了事件。

# 3 jdk的poll实现概述

jdk要做的事情就是准备参数数据，然后去调用上述poll方法，这就要用到JNI来实现。jdk使用PollSelectorImpl来实现上述poll调用。

## 3.1 pollfd参数

jdk需要将java层面接收到的一个Channel连接映射到一个pollfd结构体，PollSelectorImpl针对此创建了一个AllocatedNativeObject
对象，该对象不是在堆中，它内部使用Unsafe类直接操作内存地址。它就是专门用来存放上述一个个pollfd结构体的内容，通过固定的offset来获取每个结构体的数据内容。

所以在调用上述poll方法的时候，直接传递的是AllocatedNativeObject对象的内存地址


注册Channel要做的事：其实就是将Channel的相关数据填充到上述AllocatedNativeObject的内存地址上，下次调用poll的时候，自然就会被监控

取消Channel注册要做的事：其实就是从上述AllocatedNativeObject的内存地址上移除该Channel代表的pollfd结构体

## 3.2 nfds参数

上述注册和移除的过程，都会改变目前已注册的Channel数目，这个已注册的数目就是要传递给poll的int nfds参数


# 4 PollSelectorImpl代码分析

上述过程就可以简单理解整个实现原理，下面来详细看看具体的代码是怎么写的

## 4.1 概述

PollSelectorImpl的创建过程如下：

	PollSelectorImpl(SelectorProvider sp) {
        super(sp, 1, 1);
        long pipeFds = IOUtil.makePipe(false);
        fd0 = (int) (pipeFds >>> 32);
        fd1 = (int) pipeFds;
        pollWrapper = new PollArrayWrapper(INIT_CAP);
        pollWrapper.initInterrupt(fd0, fd1);
        channelArray = new SelectionKeyImpl[INIT_CAP];
    }

有如下2个内容

-	1 创建了pipe，得到读写文件描述符，并注册到了PollArrayWrapper中
-	2 创建了PollArrayWrapper

PollArrayWrapper pollWrapper：内部创建了一个上述介绍的AllocatedNativeObject对象（用于存放注册的Channel），而pollWrapper则更像是一个工具类，来方便的用户操作AllocatedNativeObject对象，pollWrapper把普通的操作都转化成对内存的操作，如下图所示

![PollArrayWrapper对内存的操作](https://static.oschina.net/uploads/img/201608/26094356_yj5a.png "PollArrayWrapper对内存的操作")

我们知道PollSelectorImpl在select过程的阻塞时间受控于所注册的Channel的事件，一旦有事件才会进行返回，没有事件的话就一直阻塞，为了可以允许手动控制这种局面的话，就额外增加了一个监控，即对pipe的读监控。对pipe的读文件描述符即fd0注册到PollArrayWrapper中的第一个位置，如果我们对pipe的写文件描述符fd1进行写数据操作，则pipe的读文件描述符必然会收到读事件，即可以使PollSelectorImpl不再阻塞，立即返回。

来看下初始化注册f0的代码

	void initInterrupt(int fd0, int fd1) {
        interruptFD = fd1;
        putDescriptor(0, fd0);
        putEventOps(0, POLLIN);
        putReventOps(0, 0);
    }

即将fd0存放到PollArrayWrapper的AllocatedNativeObject中，并关注POLLIN即读事件。并将pipe的写文件描述符保存到interruptFD属性中

Selector对外提供了wakeup方法，来看下PollSelectorImpl的实现

	public void interrupt() {
        interrupt(interruptFD);
    }

	private static native void interrupt(int fd);

这里就是对上述pipe的写文件描述符执行interrupt操作，来看看底层实现代码是：

	JNIEXPORT void JNICALL
	Java_sun_nio_ch_PollArrayWrapper_interrupt(JNIEnv *env, jobject this, jint fd)
	{
	    int fakebuf[1];
	    fakebuf[0] = 1;
	    if (write(fd, fakebuf, 1) < 0) {
	         JNU_ThrowIOExceptionWithLastError(env,
	                                          "Write to interrupt fd failed");
	    }
	}

这里就是简单的对pipe的写文件描述符写入数据用来触发pipe的读文件描述符的读事件而已。

至此，PollSelectorImpl的初始化过程就完成了。

## 4.2 注册和取消注册Channel过程

注册Channel其实就是向PollSelectorImpl中的PollArrayWrapper存放该Channel的fd、关注的事件信息，来看下实现代码

![注册Channel](https://static.oschina.net/uploads/img/201608/26101048_aH7M.png "注册Channel")

一点就是容量大了就需要进行对PollArrayWrapper中的AllocatedNativeObject进行扩容

另一点就是存储到PollArrayWrapper中的AllocatedNativeObject中，如下

	void addEntry(SelChImpl sc) {
        putDescriptor(totalChannels, IOUtil.fdVal(sc.getFD()));
        putEventOps(totalChannels, 0);
        putReventOps(totalChannels, 0);
        totalChannels++;
    }

存储的信息是：Channel的fd，关注的事件（初始是0）

而channel的关注事件是后来才设置到PollArrayWrapper的AllocatedNativeObject中的，见

	pollWrapper.putEventOps(sk.getIndex(), ops);

	void putEventOps(int i, int event) {
        int offset = SIZE_POLLFD * i + EVENT_OFFSET;
        pollArray.putShort(offset, (short)event);
    }

不同的Selector实现，上述实现过程也是不一样的。


再来看看取消注册Channel

![取消注册Channel](https://static.oschina.net/uploads/img/201608/26111656_eZ9C.png "取消注册Channel")

其实就是将最后一个直接覆盖到要删除的那个，以及更新相关数据的变化。

## 4.3 doSelect实现过程

如下图

![doSelect过程](https://static.oschina.net/uploads/img/201608/26112220_flPk.png "doSelect过程")

**第一步**：就是处理那些取消了的Channel,即遍历Selector的Set<SelectionKey> cancelledKeys，依次调用他们的取消注册和其他逻辑

**第二步**：就是使用pollWrapper执行poll过程，该过程即是准备好参数，然后调用linux的系统调用poll方法，如下

	int poll(int numfds, int offset, long timeout) {
        return poll0(pollArrayAddress + (offset * SIZE_POLLFD),
                     numfds, timeout);
    }

	private native int poll0(long pollAddress, int numfds, long timeout);

这里将AllocatedNativeObject的内存地址作为pollAddress，已注册的所有的Channel的数量作为numfds，timeout是用户传递的参数，然后就开始JNI调用

再看下native方法实现

	JNIEXPORT jint JNICALL
	Java_sun_nio_ch_PollArrayWrapper_poll0(JNIEnv *env, jobject this,
	                                       jlong address, jint numfds,
	                                       jlong timeout)
	{
	    struct pollfd *a;
	    int err = 0;
	
	    a = (struct pollfd *) jlong_to_ptr(address);
	
	    if (timeout <= 0) {           /* Indefinite or no wait */
	        RESTARTABLE (poll(a, numfds, timeout), err);
	    } else {                     /* Bounded wait; bounded restarts */
	        err = ipoll(a, numfds, timeout);
	    }
	
	    if (err < 0) {
	        JNU_ThrowIOExceptionWithLastError(env, "Poll failed");
	    }
	    return (jint)err;
	}

先将内存地址作为address转换成pollfd结构体地址，然后调用ipoll，在ipoll中我们就会见到linux的系统调用poll

	static int
	ipoll(struct pollfd fds[], unsigned int nfds, int timeout)
	{
	    jlong start, now;
	    int remaining = timeout;
	    struct timeval t;
	    int diff;
	
	    gettimeofday(&t, NULL);
	    start = t.tv_sec * 1000 + t.tv_usec / 1000;
	
	    for (;;) {
	        int res = poll(fds, nfds, remaining);
	        if (res < 0 && errno == EINTR) {
	            if (remaining >= 0) {
	                gettimeofday(&t, NULL);
	                now = t.tv_sec * 1000 + t.tv_usec / 1000;
	                diff = now - start;
	                remaining -= diff;
	                if (diff < 0 || remaining <= 0) {
	                    return 0;
	                }
	                start = now;
	            }
	        } else {
	            return res;
	        }
	    }
	}

至此linux系统开始为上述所有的Channel进行监控事件。

在发生了事件之后，会有2次遍历所有注册的Channel集合：

一次就是在linux底层poll调用的时候会遍历，将产生的事件值存放到pollfd结构体的revents地址中

另一次就是在java层面，获取产生的事件时，会遍历上述每一个结构体，拿到revents地址中的数据


**第三步**：一旦第二步返回就说明有事件或者超时了，一旦有事件，则linux的poll调用会把产生的事件遍历的赋值到poll调用指定的地址上，即我们指定的一个个pollfd结构体，映射到java对象就是PollArrayWrapper的AllocatedNativeObject，这时候我们获取事件就是遍历底层的每一个地址，拿到pollfd结构体中的revents，如果revents不为0代表发生了事件，还要与Channel关注的事件进行相&操作，不为0代表发生了Channel关注的事件了，并清空pollfd结构体中的revents数据供下次使用，代码如下

![获取响应事件](https://static.oschina.net/uploads/img/201608/26114117_Rf6O.png "获取响应事件")

这里就是通过指针操作直接获取对应底层结构体的revents数据。

**第四步**：上面提到了Selector也会注册一个fd用于监听，并且注册的位置时第一个即0，这里会取出该fd的发生事件，然后读取内容忽略掉即可，不然后仍然会触发该事件。代码如下

	static native boolean drain(int fd)

	JNIEXPORT jboolean JNICALL
	Java_sun_nio_ch_IOUtil_drain(JNIEnv *env, jclass cl, jint fd)
	{
	    char buf[128];
	    int tn = 0;
	
	    for (;;) {
	        int n = read(fd, buf, sizeof(buf));
	        tn += n;
	        if ((n < 0) && (errno != EAGAIN))
	            JNU_ThrowIOExceptionWithLastError(env, "Drain");
	        if (n == (int)sizeof(buf))
	            continue;
	        return (tn > 0) ? JNI_TRUE : JNI_FALSE;
	    }
	}


# 5 结束语

下一篇就来详细介绍下jdk与linux的epoll对接实现