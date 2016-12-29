本文针对jdk1.8的ConcurrentHashMap

# 1 1.8的HashMap设计

## 1.1 整体概览

HashMap采用的是**数组+链表+红黑树**的形式。

数组是可以扩容的，链表也是转化为红黑树的，这2种方式都可以承载更多的数据。

用户可以设置的参数：初始总容量默认16，默认的加载因子0.75

初始的数组个数默认是16（用户不能设置的）

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

## 1.1 整体概览

ConcurrentHashMap有3个参数：

-	initialCapacity：初始总容量，默认16
-	loadFactor：加载因子，默认0.75
-	concurrencyLevel：并发级别，默认16

然后我们需要知道的是：

-	segment的个数即ssize

	取大于等于并发级别的最小的2的指数。如concurrencyLevel=16，那么sszie=16,如concurrencyLevel=10，那么ssize=16

-	单个segment的初始容量cap

	c=initialCapacity/ssize,并且可能需要+1。如15/7=2，那么c要取3，如16/8=2，那么c取2

	c可能是一个任意值，那么同上述一样，cap取的值就是大于等于c的最下2的指数。最小值要求是2

-	单个segment的阈值threshold

	cap*loadFactor

所以默认情况下，segment的个数sszie=16,每个segment的初始容量cap=2，单个segment的阈值threshold=1

## 1.2 put过程

-	首先根据key计算出一个hash值，找到对应的Segment
-	调用Segment的lock方法，为后面的put操作加锁
-	根据key计算出hash值，找到Segment中数组中对应index的链表，并将该数据放置到该链表中
-	判断当前Segment包含元素的数量大于阈值，则Segment进行扩容

整体代码逻辑见如下源码：

![1.7ConcurrentHashMap的put过程1](https://static.oschina.net/uploads/img/201612/29161537_qIUR.png "1.7ConcurrentHashMap的put过程1")

其中上述Segment的put过程源码如下：

![1.7ConcurrentHashMap的put过程2](https://static.oschina.net/uploads/img/201612/29161907_hI3n.png "1.7ConcurrentHashMap的put过程2")

## 1.3 扩容过程

这个扩容是在Segment的锁的保护下进行扩容的，不需要关注并发问题。

![Segment扩容如下所示](https://static.oschina.net/uploads/img/201612/29164040_Me8a.png "Segment扩容如下所示")

这里的重点就是：

首先找到一个lastRun，lastRun之后的元素和lastRun是在同一个桶中，所以后面的不需要进行变动。

然后对开始到lastRun部分的元素，重新计算下设置到newTable中，每次都是将当前元素作为newTable的首元素，之前老的链表作为该首元素的next部分。

## 1.4 get过程

-	根据key计算出对应的segment
-	再根据key计算出对应segment中数组的index
-	最终遍历上述index位置的链表，查找出对应的key的value

源码如下：

![1.7ConcurrentHashMap的get过程](https://static.oschina.net/uploads/img/201612/29164827_4mOm.png "1.7ConcurrentHashMap的get过程")

# 3 1.8的ConcurrentHashMap设计

1.8的ConcurrentHashMap摒弃了1.7的设计，而是在1.8HashMap的基础上实现了线程安全的版本，即也是采用**数组+链表+红黑树**的形式。

数组可以扩容，链表可以转化为红黑树

## 3.1 整体概览

有一个重要的参数sizeCtl，代表数组的大小（但是还有其他取值及其含义，后面再详细说到）

用户可以设置一个初始容量initialCapacity给ConcurrentHashMap

sizeCtl=大于（1.5倍initialCapacity+1）的最小的2的指数。

即initialCapacity=20，则sizeCtl=32,如initialCapacity=24，则sizeCtl=64。

初始化的时候，会按照sizeCtl的大小创建出对应大小的数组

## 3.2 put过程

源码如下所示：

![1.8ConcurrentHashMap的put过程](https://static.oschina.net/uploads/img/201612/29180141_DUUU.png "1.8ConcurrentHashMap的put过程")




## 3.3 get过程

## 3.4 扩容过程

# 4 问题分析

## 4.1 ConcurrentHashMap读为什么不需要锁？


## 4.2 