# 1系列内容

-	jdk Selector设计情况
-	jdk nio select、poll在linux平台下的实现
-	jdk nio epoll在linux平台下的实现
-	netty 原生epoll在linux平台下的实现
-	epoll的2种通知模式边缘触发、水平触发

# 2 jdk Selector概述

首先来看下文档描述

## 2.1 创建方式

一个就是直接调用open方法

	public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

或者调用选用某个SelectorProvider的openSelector

	public abstract AbstractSelector openSelector()

## 2.2 SelectionKey

一个Selector有3种SelectionKey集合

-	一种就是全部注册的SelectionKey集合，即keys()方法返回的结果

-	一种就是活跃的SelectionKey集合，即selectedKeys()方法返回的结果

-	第三种就是已取消的SelectionKey集合，这些已被取消但是还未从Selector取消注册

再确认下下面的几个问题：

-	什么叫注册

	即将一个channel感兴趣的事件注册到Selector上，即如下方法

		SelectionKey register(AbstractSelectableChannel ch, int ops, Object att)

	对于select、poll的Selector实现：仅仅是保存上述AbstractSelectableChannel感兴趣的ops到指定地方

	对于epoll的Selector实现：则是执行系统调用epoll_ctl方法，操作参数是EPOLL_CTL_ADD

-	什么叫取消

	就是调用SelectionKey的cancel方法，该方法的实现是，将该SelectionKey放入cancelledKeys而已，没有做其他操作

-	什么叫取消注册

	取消注册，就是从Selector从释放出来不再关注某个事件。通常在我们的select过程中就会遍历上述cancelledKeys，依次执行取消注册的行为。不同的Selector有不同的行为，如epoll则是执行系统调用epoll_ctl方法，操作参数是EPOLL_CTL_DEL。

## 2.3 三种select方式

分别如下：

	int select()：表示一直阻塞到有事件为止

	int select(long timeout)：最多阻塞timeout时间

	int selectNow()：不阻塞，检查结果后立即返回

## 2.4 并发性

Selector不是线程安全的，不过大部分情况都是一个线程拥有一个Selector，所以不需要它线程安全。

## 2.5 使用案例

以ZooKeeper为例，代码简述如下

	final Selector selector = Selector.open();

	public void run() {
        while (!ss.socket().isClosed()) {
            try {
                selector.select(1000);

                Set<SelectionKey> selected = selector.selectedKeys();

                for (SelectionKey k : selected) {
                    if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {

						//执行accept连接的事件

                        SocketChannel sc = ((ServerSocketChannel) k
                                .channel()).accept();
                        sc.configureBlocking(false);
                        SelectionKey sk = sc.register(selector,
                                SelectionKey.OP_READ);
                        NIOServerCnxn cnxn = createConnection(sc, sk);
                        sk.attach(cnxn);
                        addCnxn(cnxn);
                    } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
						
						//执行IO读写事件
						
                        NIOServerCnxn c = (NIOServerCnxn) k.attachment();
                        c.doIO(k);
                    } else {
                        //未知
                    }
                }
                selected.clear();
            } catch (RuntimeException e) {
                LOG.warn("Ignoring unexpected runtime exception", e);
            } catch (Exception e) {
                LOG.warn("Ignoring exception", e);
            }
        }
        closeAll();
        LOG.info("NIOServerCnxn factory exited run method");
    }

while循环里面不断调用selector.select(1000)方法，然后通过selector.selectedKeys()来获取到有事件的SelectionKey集合，即上述提到的活跃的SelectionKey集合。然后遍历该集合，执行对应的事件。

# 3 不同实现类

目前Selector目前有如下实现

![Selector实现类](https://static.oschina.net/uploads/img/201608/16173917_ZNe8.png "Selector实现类")

针对linux平台的实现：

-	PollSelectorImpl：基于poll来实现
-	EPollSelectorImpl：基于epoll来实现

针对windows平台的实现：

-	WindowsSelectorImpl

而Netty的NioEventLoop则是使用上述linux平台的实现PollSelectorImpl。Netty自己提供了另外一种epoll实现，没有直接采用上述jdk自带的EPollSelectorImpl。

# 4 后续

接下来就要开始重点说说jdk的poll是怎么实现的，即PollSelectorImpl的内容