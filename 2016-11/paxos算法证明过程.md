paxos算法有运作过程和证明过程，运作过程比较清晰明了，但是证明过程就比较复杂了。

很多人能够看懂paxos算法的运行过程，分prepare过程和accept过程，但是总是对证明过程模模糊糊，或者在看证明过程时和运作过程相混淆，特别是下文中的P2c证明P2b的过程，可能会犯下拿运作过程当做已知条件来证明证明过程的错误。所以要想理解证明过程，必须先抛弃运作过程的认知，特别是下面的1-6点都是针对accept过程的，还不存在prepare过程，prepare过程是被逐步引导出来的。

一开始场景只有如下描述

	proposer发出一个议案，可能被某些acceptor accept，只要被足够多的acceptor accept则意味着该议案被chosen，同时该议案的value被chosen

以及我们想要达到的目标：

	只能有一个value能被chosen

议案和value的关系，多个议案可能包含同一个value。那么只有一个议案能被chosen还是多个？目前还是未知。

下面先来看第一个问题：足够多的

# 1 足够多的问题

这里我们可能不自觉的就说过半，但是感觉又说不上来为什么要过半

我们先来总结下过半：

-	使用场景：在要求**每个server只能认同一个结论，不能认同2个及其以上的结论**的场景下达成一致的一个结论
-	为什么：一旦过半的server认同结论A，则在上述场景下，不可能再出现过半的server认同结论B。如果出现了，说明必有一个server认同了结论A又认同了结论B，所以违反了上述场景要求，所以在上述场景下是不可能出现的。

所以只有在上述场景下才会采用过半来达成一致。而我们上述paxos的value被chosen就是这样的一个场景，一个server只能chosen一个value，不能chosen2个及其以上的value，所以上述的足够多就可以来使用过半

目前场景描述为：

	发出一个议案，可能被某些acceptor accept，只要被过半的acceptor accept则意味着该议案被chosen，同时该议案的value被chosen

接下来就要对该场景进行细致的展开，先来说说acceptor的accept要求

# 2 acceptor的初始accept

引出了P1，如下

-	P1. An acceptor must accept the first proposal that it receives

	一个acceptor必须accept它接收的第一个议案

P1就面临了一个抉择问题，一个acceptor还能不能accept其他议案？即acceptor是否允许accept多个议案？

-	如果不能accept多个议案，则很可能无法形成过半，目前弃用。

	Raft则是只能accept一个议案，一旦无法形成多数，则开启下一轮的accept，通过随机延时来消除这种情况

-	如果可以accept多个议案，又要保证我们的目标：只能有一个value被chosen。这就引出了如下P2要求来做到我们的目标


# 3 P2-对结果要求

P2要求如下：

-	P2：If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v

	如果一个value为v的议案被chosen了，则更高的议案必须含有value v才能被chosen。

有了这个约束，我们就能保证在多个议案被chosen的情况下，只有一个value被chosen。

P2更像是对chosen一个议案的要求，如果要想实现它，必须把它落实在acceptor的accept上，那就引出了下面的P2a的要求

# 4 P2a-对acceptor的accept要求

-	P2a： If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v

	如果一个value为v的议案被chosen，那么每一个acceptor accept的更高议案是必须含有value v的

acceptor aceept的高议案都是含有value v的，则这些高议案被chosen的时候自然满足P2。

这时候P1是存在另一个问题。如当一个议案被chosen的过程中，server c由于网络原因没有收到上述议案的。之后议案被chosen，如果有另一个新的proposer发起了一个更高的议案同时带上一个不同的value v1，该议案发往server c（没有收到任何议案），根据P1，server c是必须要accept该议案的。也就是说在一个议案被chosen的情况下，该acceptor accept的更高议案的value却不是v，即和P2a是矛盾的。

为了同时保证P1和P2a，我们需要对P2a进一步的限制，就引出了P2b

# 5 P2b-对proposer提出议案的要求（结果上要求）

-	P2b：If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v

	如果一个value为v的议案被chosen，那么如果一个proposer要提出一个更高的议案，则该议案必须含有value v。

这样的话就杜绝了上述情况，在value v被chosen的情况下，proposer要想提出一个议案，必须采用之前已提交的value v，而不是使用自己的value v1。同时又保证了P2a。

P2b对proposer提出的议案做出了要求，但是这个议案怎么来（如怎么得到已经被chosen的value v）并没有说明，下面就引出了P2c，来详细描述proposer通过怎样的过程提出的议案满足上面的要求。

# 6 P2c-对proposer提出议案的要求（做法上要求）

-	P2c：For any v and n, if a proposal with value v and number n is issued,then there is a set S consisting of a majority of acceptors such that either 

	(a) no acceptor in S has accepted any proposal numbered less than n, 

	or (b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S

	要想提出一个议案号为n，value为v的议案，必须满足下面2个条件中的一个

	S是一个过半的acceptor集合，满足如下2个条件中的一个

	-	a：S中的acceptor都没有accept任何小于n的议案（相当于初始时即acceptor还没开始accept议案，此时可以随意提议案）
	-	b：S中的acceptor有accept的小于n的议案，则v是上述议案中编号最大的议案的value

比较难以理解的就是P2c给出的提出议案的方案如何能保证P2b。再来看看P2b要证明的是：在一个value为v的议案被chosen的情况下，保证新提出的议案必须含有value v，结合P2c对提出议案的要求，a条件被排除了（因为在P2b的条件下已经有议案被accept了），那就是说目前要在b条件下证明P2b。

即目前的证明题是：

**如果一个议案号为m，value为v的议案被chosen了，在满足b的条件下，提出一个议案（n，v1），证明v1=v**

这个证明可以用归纳法来证明。

-	第一步：证明初值成立，如果n=m+1，则至少过半集合C中的acceptor中小于n的最大accept议案是m，m的value是v，根据b条件，S中必然有一个acceptor属于C，即S中必然有一个acceptor accept的最大议案就是m，m已经是最大议案，即S集合中最大accept议案必然是m，所以此时新议案选用的value就是m的value v。
-	第二步：假设m到n-1的议案的value都是v（m之后的议案是否被chosen处于未知状态，至少m议案是被过半accept的），此时过半的集合C中的acceptor accept的最大议案必然落在[m，n-1]中，他们的value全部是v，根据b条件，S中必然有一个或者一些acceptor是属于C集合的，S和C之间公共的acceptor集合为C1，则C1集合具有上述C集合的特点，S中的acceptor accept的最大议案必然落在C1中，由于C1中m之后的议案的value全是v，则此时提出的新议案的value必然采用value v。

总上2步，P2c给出的这个提出议案的要求，必然能够满足P2b。

接下来的问题就是：一个proposer如何知道acceptor accept的最大议案的value呢？这就需要proposer先提前去探测下这个最大议案的value，即这时候才引出运作过程中的prepare过程。前面一直在说的是运作过程的accept过程。

# 7 引出prepare过程和P1a

一个proposer向所有的acceptor发送请求，包含要发送的议案号n，来得知他们当前accept的最大议案的value。该请求就被称为prepare请求

这些acceptor的议案可以分成2大部分：

-	7.1 小于议案号n的议案

	又细分成2部分

	-	7.1.1 目前已经被accept的议案

	从中可以挑选出最大accept的议案的value，作为该proposer要提出的议案的value
	-	7.1.2 还未被accept的议案

	这部分议案是还未到达acceptor，proposer要做的就是不再这些议案全部到达被accept了之后再去选择其中最大议案的value，而是直接让acceptor保证：抛弃这一部分的议案，即承诺此后不再accept 小于n的议案了。从而简化对这部分议案的处理。

	这一部分约束就是对acceptor的accept约束的补充，即

	P1 a . An acceptor can accept a proposal numbered n if it has not responded to a prepare request having a number greater than n

	如果一个acceptor没有对大于n的议案的prepare请求进行响应的前提下，该acceptor才能accept议案n，否则就要拒绝议案n。
	
-	7.2 大于议案号n的议案

	如果acceptor accept了大于n的议案，从中选举最大议案的value，作为该proposer要提出的议案的value（这一步其实是没有必要的，后面会优化这一步）

# 8 优化prepare

对于上述7.2的情况，即proposer一旦发出了一个prepare请求，议案编号为n，如果此时acceptor 已经accept了更大的议案，如n+1。

则由上述7.1.2可以得知，在收到n+1的议案的prepare的时候，已经做出了承诺，不再accept小于n+1的议案了，所以该proposer提出的编号为n的议案即使在prepare过程中得到了value,该议案在发给acceptor accept的时候，acceptor是拒绝accept的。

所以应该在prepare阶段就可以把它拒绝了，所以7.2进一步更改为：

-	7.2 大于议案号n的议案

	直接拒绝proposer发送的议案号为n的prepare请求。


我们一直在说acceptor对于prepare有2个承诺一个应答，其实就是上述7的三个分支。

-	7.1.1分支是应答
-	7.1.2承诺不再accept小于n的议案
-	7.2 承诺不再响应小于n的prepare请求

# 9 paxos的整体运作过程

-	Phase 1：

	-	(a) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors

		一个proposer选择一个编号为n的议案，向所有的acceptor发送prepare请求

	-	(b) If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded,then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered proposal (if any) that it has accepted

		如果acceptor已经响应的prepare请求中议案编号都比n小，则它承诺不再响应prepare请求或者accept请求中议案编号小于n的，
		并且找出已经accept的最大议案的value返回给该proposer

		如果已响应的编号比n大，则直接忽略该prepare请求。

-	Phase 2：

	-	(a) If the proposer receives a response to its prepare requests (numbered n) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v, where v is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals

		如果proposer收到了过半的acceptors响应，那么将提出一个议案（n，v）,v就是上述所有acceptor响应中最大accept议案的value，或者是proposer自己的value。然后将该议案发送给所有的acceptor。

		这个请求叫做accept请求，这一步才是所谓发送议案请求，而前面的prepare请求更多的是一个构建出最终议案(n,v)的过程。

		如果没有收到过半响应，则增大议案编号，重新回到Phase 1阶段。

	-	(b) If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n

		acceptor接收到编号为n的议案，如果acceptor还没有对大于n的议案的prepare请求响应过，则acceptor就accept该议案，否则拒绝。

# 10 base paxos和multi-paxos

上述过程全部说的是一轮paxos的执行过程最终选出一个议案达成一致性，属于base paxos，而对于多轮paxos即multi-paxos。multi-paxos又会很多的问题，这个之后再详细说明。


**欢迎继续来讨论，越辩越清晰**。

欢迎关注微信公众号：乒乓狂魔

![乒乓狂魔微信公众号](https://static.oschina.net/uploads/img/201610/28090041_LsUp.png "乒乓狂魔微信公众号")