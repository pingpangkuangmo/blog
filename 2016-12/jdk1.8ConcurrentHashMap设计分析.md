本文针对jdk1.8的ConcurrentHashMap

# 1 1.8的HashMap设计

## 1.1 整体概览

HashMap采用的是**数组+链表+红黑树**的形式。

数组是可以扩容的，链表也是转化为红黑树的，这2种方式都可以承载更多的数据。

用户可以设置的参数：初始总容量默认16，默认的加载因子0.75

初始的数组个数默认是16

容量X加载因子=阈值

一旦目前容量超过该阈值，则执行扩容操作。

什么时候扩容？

-	1 当前容量超过阈值
-	2 当链表中元素个数超过默认设定（8个），当数组的大小还未超过64的时候，此时进行数组的扩容，如果超过则将链表转化成红黑树

什么时候链表转化为红黑树？（上面已经提到了）

-	当数组大小已经超过64并且链表中的元素个数超过默认设定（8个）时，将链表转化为红黑树


目前形象的表示数组中的一个元素称为一个桶

## 1.2 put过程

-	根据key计算出hash值
-	hash值&（数组长度-1）得到所在数组的index
	-	如果该index位置的Node元素不存在，则直接创建一个新的Node
	-	如果该index位置的Node元素是TreeNode类型即红黑树类型了，则直接按照红黑树的插入方式进行插入
	-	如果该index位置的Node元素是非TreeNode类型则，则按照链表的形式进行插入操作

		链表插入操作完成后，判断是否超过阈值TREEIFY_THRESHOLD（默认是8），超过则要么数组扩容要么链表转化成红黑树
-	判断当前总容量是否超出阈值，如果超出则执行扩容

源码如下：

![HashMap的put过程](https://static.oschina.net/uploads/img/201612/28154542_9bV6.png "HashMap的put过程")

下面来说说这个扩容的过程

## 1.3 扩容过程

按照2倍扩容的方式，那么就需要将之前的所有元素全部重新按照2倍桶的长度重新计算所在桶。这里为啥是2倍？

因为2倍的话，更加容易计算他们所在的桶，并且各自不会相互干扰。如原桶长度是4，现在桶长度是8，那么

-	桶0中的元素会被分到桶0和桶4中
-	桶1中的元素会被分到桶1和桶5中
-	桶2中的元素会被分到桶2和桶6中
-	桶3中的元素会被分到桶3和桶7中

为啥是这样呢？

桶0中的元素的hash值后2位必然是00，这些hash值可以根据后3位000或者100分成2类数据。他们分别&（8-1）即&111,则后3位为000的在桶0中，后3位为100的必然在桶4中。其他同理，也就是说桶4和桶0重新瓜分了原来桶0中的元素。

如果换成其他倍数，那么瓜分就比较混乱了。

这样在瓜分这些数据的时候，只需要先把这些数据分类，如上述桶0中分成000和100 2类，然后直接构成新的链表，分类完毕后，直接将新的链表挂在对应的桶下即可，源码如下：

![HashMap的扩容过程](https://static.oschina.net/uploads/img/201612/28161812_Tk31.png "HashMap的扩容过程")

上述 (e.hash & oldCap) == 0 即可将原桶中的数据分成2类

上述是对于链表情况下的重新移动，而针对红黑树情况下：

则需要考虑分类之后是否还需要依然保持红黑树，如果个数少则直接使用链表即可。

## 1.4 get过程

get过程比较简单

-	根据key计算出hash值
-	hash值&（数组长度-1）得到所在数组的index

	-	如果要找的key就是上述数组index位置的元素，直接返回该元素的值
	-	如果该数组index位置元素是TreeNode类型，则按照红黑树的查询方式来进行查找
	-	如果该数组index位置元素非TreeNode类型，则按照链表的方式来进行遍历查询

源码如下：

![HashMap的get过程](https://static.oschina.net/uploads/img/201612/28165110_Qgbu.png "HashMap的get过程")

# 2 1.7的ConcurrentHashMap设计

ConcurrentHashMap是线程安全，通过分段锁的方式提高了并发度。分段是一开始就确定的了，后期不能再进行扩容的。

其中的段Segment继承了重入锁ReentrantLock，有了锁的功能，同时含有类似HashMap中的数组加链表结构（这里没有使用红黑树）

虽然Segment的个数是不能扩容的，但是单个Segment里面的数组是可以扩容的。

## 2.1 整体概览

ConcurrentHashMap有3个参数：

-	initialCapacity：初始总容量，默认16
-	loadFactor：加载因子，默认0.75
-	concurrencyLevel：并发级别，默认16

然后我们需要知道的是：

-	segment的个数即ssize

	取大于等于并发级别的最小的2的幂次。如concurrencyLevel=16，那么sszie=16,如concurrencyLevel=10，那么ssize=16

-	单个segment的初始容量cap

	c=initialCapacity/ssize,并且可能需要+1。如15/7=2，那么c要取3，如16/8=2，那么c取2

	c可能是一个任意值，那么同上述一样，cap取的值就是大于等于c的最下2的幂次。最小值要求是2

-	单个segment的阈值threshold

	cap*loadFactor

所以默认情况下，segment的个数sszie=16,每个segment的初始容量cap=2，单个segment的阈值threshold=1

## 2.2 put过程

-	首先根据key计算出一个hash值，找到对应的Segment
-	调用Segment的lock方法，为后面的put操作加锁
-	根据key计算出hash值，找到Segment中数组中对应index的链表，并将该数据放置到该链表中
-	判断当前Segment包含元素的数量大于阈值，则Segment进行扩容

整体代码逻辑见如下源码：

![1.7ConcurrentHashMap的put过程1](https://static.oschina.net/uploads/img/201612/29161537_qIUR.png "1.7ConcurrentHashMap的put过程1")

其中上述Segment的put过程源码如下：

![1.7ConcurrentHashMap的put过程2](https://static.oschina.net/uploads/img/201612/29161907_hI3n.png "1.7ConcurrentHashMap的put过程2")

## 2.3 扩容过程

这个扩容是在Segment的锁的保护下进行扩容的，不需要关注并发问题。

![Segment扩容如下所示](https://static.oschina.net/uploads/img/201612/29164040_Me8a.png "Segment扩容如下所示")

这里的重点就是：

首先找到一个lastRun，lastRun之后的元素和lastRun是在同一个桶中，所以后面的不需要进行变动。

然后对开始到lastRun部分的元素，重新计算下设置到newTable中，每次都是将当前元素作为newTable的首元素，之前老的链表作为该首元素的next部分。

## 2.4 get过程

-	根据key计算出对应的segment
-	再根据key计算出对应segment中数组的index
-	最终遍历上述index位置的链表，查找出对应的key的value

源码如下：

![1.7ConcurrentHashMap的get过程](https://static.oschina.net/uploads/img/201612/29164827_4mOm.png "1.7ConcurrentHashMap的get过程")

# 3 1.8的ConcurrentHashMap设计

1.8的ConcurrentHashMap摒弃了1.7的segment设计，而是在1.8HashMap的基础上实现了线程安全的版本，即也是采用**数组+链表+红黑树**的形式。

数组可以扩容，链表可以转化为红黑树

## 3.1 整体概览

有一个重要的参数sizeCtl，代表数组的大小（但是还有其他取值及其含义，后面再详细说到）

用户可以设置一个初始容量initialCapacity给ConcurrentHashMap

sizeCtl=大于（1.5倍initialCapacity+1）的最小的2的幂次。

即initialCapacity=20，则sizeCtl=32,如initialCapacity=24，则sizeCtl=64。

初始化的时候，会按照sizeCtl的大小创建出对应大小的数组

## 3.2 put过程

源码如下所示：

![1.8ConcurrentHashMap的put过程](https://static.oschina.net/uploads/img/201612/29180141_DUUU.png "1.8ConcurrentHashMap的put过程")

-	如果数组还未初始化，那么进行初始化，这里会通过一个CAS操作将sizeCtl设置为-1，设置成功的，可以进行初始化操作
-	根据key的hash值找到对应的桶，如果桶还不存在，那么通过一个CAS操作来设置桶的第一个元素，失败的继续执行下面的逻辑即向桶中插入或更新
-	如果找到的桶存在，但是桶中第一个元素的hash值是-1，说明此时该桶正在进行迁移操作，这一块会在下面的扩容中详细谈及。
-	如果找到的桶存在，那么要么是链表结构要么是红黑树结构，此时需要获取该桶的锁，在锁定的情况下执行链表或者红黑树的插入或更新

	-	如果桶中第一个元素的hash值大于0，说明是链表结构，则对链表插入或者更新
	-	如果桶中的第一个元素类型是TreeBin，说明是红黑树结构，则按照红黑树的方式进行插入或者更新
-	在锁的保护下插入或者更新完毕后，如果是链表结构，需要判断链表中元素的数量是否超过8（默认），一旦超过就要考虑进行数组扩容或者是链表转红黑树

下面就来重点看看这个扩容过程

## 3.3 扩容过程

一旦链表中的元素个数超过了8个，那么可以执行数组扩容或者链表转为红黑树，这里依据的策略跟HashMap依据的策略是一致的。

当数组长度还未达到64个时，优先数组的扩容，否则选择链表转为红黑树。

源码如下所示：

![扩容或者链表转红黑树](https://static.oschina.net/uploads/img/201612/30174822_IGdO.png "扩容或者链表转红黑树")

重点来看看这个扩容过程，即看下上述tryPresize方法，也可以看到上述是2倍扩容的方式

![输入图片说明](https://static.oschina.net/uploads/img/201701/02214931_DmPC.png "在这里输入图片标题")

第一个执行的线程会首先设置sizeCtl属性为一个负值，然后执行transfer(tab, null)，其他晚进来的线程会检查当前扩容是否已经完成，没完成则帮助进行扩容，完成了则直接退出。

该ConcurrentHashMap的扩容操作可以允许多个线程并发执行，那么就要处理好任务的分配工作。每个线程获取一部分桶的迁移任务，如果当前线程的任务完成，查看是否还有未迁移的桶，若有则继续领取任务执行，若没有则退出。在退出时需要检查是否还有其他线程在参与迁移工作，如果有则自己什么也不做直接退出，如果没有了则执行最终的收尾工作。

-	问题1：当前线程如何感知其他线程也在参与迁移工作？

	靠sizeCtl的值，它初始值是一个负值=(rs << RESIZE_STAMP_SHIFT) + 2)，每当一个线程参与进来执行迁移工作，则该值进行CAS自增，该线程的任务执行完毕要退出时对该值进行CAS自减操作，所以当sizeCtl的值等于上述初值则说明了此时未有其他线程还在执行迁移工作，可以去执行收尾工作了。见如下代码

	![部分迁移工作结束判断](https://static.oschina.net/uploads/img/201701/03102615_o8GS.png "部分迁移工作结束判断")

-	问题2：任务按照何规则进行分片？

	![输入图片说明](https://static.oschina.net/uploads/img/201701/02221345_zLYi.png "在这里输入图片标题")

	上述stride即是每个分片的大小，目前有最低要求16，即每个分片至少需要16个桶。stride的计算依赖于CPU的核数，如果只有1个核，那么此时就不用分片，即stride=n。其他情况就是 (n >>> 3) / NCPU。

-	问题3：如何记录目前已经分出去的任务？

	ConcurrentHashMap含有一个属性transferIndex（初值为最后一个桶），表示从transferIndex开始到后面所有的桶的迁移任务已经被分配出去了。所以每次线程领取扩容任务，则需要对该属性进行CAS的减操作，即一般是transferIndex-stride。

-	问题4：每个线程如何处理分到的部分桶的迁移工作

	第一个获取到分片的线程会创建一个新的数组，容量是之前的2倍。

	遍历自己所分到的桶：

	-	桶中元素不存在，则通过CAS操作设置桶中第一个元素为ForwardingNode，其Hash值为MOVED（-1）,同时该元素含有新的数组引用

		此时若其他线程进行put操作，发现第一个元素的hash值为-1则代表正在进行扩容操作（并且表明该桶已经完成扩容操作了，可以直接在新的数组中重新进行hash和插入操作），该线程就可以去参与进去，或者没有任务则不用参与，此时可以去直接操作新的数组了

	-	桶中元素存在且hash值为-1，则说明该桶已经被处理了（本不会出现多个线程任务重叠的情况，这里主要是该线程在执行完所有的任务后会再次进行检查，再次核对）

	-	桶中为链表或者红黑树结构，则需要获取桶锁，防止其他线程对该桶进行put操作，然后处理方式同HashMap的处理方式一样，对桶中元素分为2类，分别代表当前桶中和要迁移到新桶中的元素。设置完毕后代表桶迁移工作已经完成，旧数组中该桶可以设置成ForwardingNode了

下面来看下详细的代码：

![扩容代码](https://static.oschina.net/uploads/img/201701/03103705_Nzg1.png "扩容代码")

## 3.4 get过程

-	根据k计算出hash值，找到对应的数组index
-	如果该index位置无元素则直接返回null
-	如果该index位置有元素

	-	如果第一个元素的hash值小于0，则该节点可能为ForwardingNode或者红黑树节点TreeBin

		如果是ForwardingNode（表示当前正在进行扩容），使用新的数组来进行查找
		
		如果是红黑树节点TreeBin，使用红黑树的查找方式来进行查找

	-	如果第一个元素的hash大于等于0，则为链表结构，依次遍历即可找到对应的元素

详细代码如下

![1.8ConcurrentHashMap的get过程](https://static.oschina.net/uploads/img/201701/03115030_wxIm.png "1.8ConcurrentHashMap的get过程")

至此，ConcurrentHashMap主要的操作都粗略的介绍完毕了，其他一些操作靠各位自行去看了。

下面针对一些问题来进行解答

# 4 问题分析

## 4.1 ConcurrentHashMap读为什么不需要锁？

我们通常使用读写锁来保护对一堆数据的读写操作。读时加读锁，写时加写锁。在什么样的情况下可以不需要读锁呢？

如果对数据的读写是一个原子操作，那么此时是可以不需要读锁的。如ConcurrentHashMap对数据的读写，写操作是不需要分2次写的（没有中间状态），读操作也是不需要2次读取的。假如一个写操作需要分多次写，必然会有中间状态，如果读不加锁，那么可能就会读到中间状态，那就不对了。

假如ConcurrentHashMap提供put(key1,value1,key2,value2)，写入的时候必然会存在中间状态即key1写完成，但是key2还未写，此时如果读不加锁，那么就可能读到key1是新数据而key2是老数据的中间状态。

虽然ConcurrentHashMap的读不需要锁，但是需要保证能读到最新数据，所以必须加volatile。即数组的引用需要加volatile，同时一个Node节点中的val和next属性也必须要加volatile。

## 4.2 ConcurrentHashMap是否可以在无锁的情况下进行迁移？

目前1.8的ConcurrentHashMap迁移是在锁定旧桶的前提下进行迁移的，然而并没有去锁定新桶。那么就可能提出如下问题：

-	在某个桶的迁移过程中，别的线程想要对该桶进行put操作怎么办？

	一旦某个桶在迁移过程中了，必然要获取该桶的锁，所以其他线程的put操作要被阻塞，一旦迁移完毕，该桶中第一个元素就会被设置成ForwardingNode节点，所以其他线程put时需要重新判断下桶中第一个元素是否被更改了，如果被改了重新获取重新执行逻辑，如下代码

	![迁移过程中的阻塞](https://static.oschina.net/uploads/img/201701/03152150_CxU7.png "迁移过程中的阻塞")

-	某个桶已经迁移完成（其他桶还未完成），别的线程想要对该桶进行put操作怎么办？

	该线程会首先检查是否还有未分配的迁移任务，如果有则先去执行迁移任务，如果没有即全部任务已经分发出去了，那么此时该线程可以直接对新的桶进行插入操作（映射到的新桶必然已经完成了迁移，所以可以放心执行操作）

从上面看到我们在迁移的时候还是需要对旧桶锁定的，能否在无锁的情况下实现迁移？

可以参考参考这篇论文[Split-Ordered Lists: Lock-Free Extensible Hash Tables](http://people.csail.mit.edu/shanir/publications/Split-Ordered_Lists.pdf)

一旦扩容就涉及到迁移桶中元素的操作，将一个桶中的元素迁移到另一个桶中的操作不是一个原子操作，所以需要在锁的保护下进行迁移。如果扩容操作是移动桶的指向，那么就可以通过一个CAS操作来完成扩容操作。上述Split-Ordered Lists就是把所有元素按照一定的顺序进行排列。该list被分成一段一段的，每一段都代表某个桶中的所有元素。每个桶中都有一个指向第一个元素的指针，如下图结构所示：

![Split-Ordered Lists](https://static.oschina.net/uploads/img/201701/03160126_OFsv.png "Split-Ordered Lists")

每一段其实也是分成2类的，如同前面所说的HashMap在扩容是分成2类的情况是一样的，此时Split-Ordered Lists在扩容时就只需要将新桶的指针指向这2类的分界点即可。

这一块之后再详细说明吧。

## 4.3 ConcurrentHashMap曾经的弱一致性

具体详见这篇针对老版本的ConcurrentHashMap的说明文章[为什么ConcurrentHashMap是弱一致的](http://ifeve.com/concurrenthashmap-weakly-consistent/)

文中已经解释到：对数组的引用是volatile来修饰的，但是数组中的元素并不是。即读取数组的引用总是能读取到最新的值，但是读取数组中某一个元素的时候并不一定能读到最新的值。所以说是弱一致性的。

我觉得这个只需要稍微改动下就可以实现强一致性：

-	对于新加的key，通过写入到链表的末尾即可。因为一个元素的next属性是volatile的，可以保证写入后立马看的到，如下1.8的方式

-	或者对数组中元素的更新采用volatile写的方式，如下1.7的形式

但是现在1.7版本的ConcurrentHashMap对于数组中元素的写也是加了volatile的，如下代码

![1.7版本的ConcurrentHashMap对于数组元素的写](https://static.oschina.net/uploads/img/201701/03150220_mPOj.png "1.7版本的ConcurrentHashMap对于数组元素的写")

1.8的方式就是：直接将新加入的元素写入next属性（含有volatile修饰）中而不是修改桶中的第一个元素。

![1.8版本的ConcurrentHashMap的volatile写](https://static.oschina.net/uploads/img/201701/03151112_2IxR.png "1.8版本的ConcurrentHashMap的volatile写")

所以在1.7和1.8版本的ConcurrentHashMap中不再是弱一致性，写入的数据是可以立马本读到的。

# 5 结束语

本篇文章因为涉及的话题众多，难免有疏漏之处，还请指出纠正。更多的问题可以加群讨论，详见这篇加群说明文章[加群指南](https://my.oschina.net/pingpangkuangmo/blog/799041)

**欢迎继续来讨论，越辩越清晰**。

欢迎关注微信公众号：乒乓狂魔

![乒乓狂魔微信公众号](https://static.oschina.net/uploads/img/201610/28090041_LsUp.png "乒乓狂魔微信公众号")