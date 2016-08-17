# 1系列内容

-	jdk Selector设计情况
-	jdk nio select、poll在linux平台下的实现
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

整体的意思就是：你指定了结构体列表的起始地址和要监控的结构体个数，linux系统就会为你在timeout事件内监控上述结构体列表中的文件描述符的相关事件，并把事件写入到上述的short revents属性中。

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


## 4.1 属性概述

## 4.2 注册和取消注册Channel过程

## 4.3 doSelect实现过程
