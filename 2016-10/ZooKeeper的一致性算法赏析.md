# 1 ZAB协议介绍

ZAB协议是。。。，分成如下4个阶段。

## 1.1 概念介绍

peerEpoch、electionEpoch

## 1.1 Leader election

## 1.2 Discovery

## 1.3 Synchronization

## 1.4 Broadcast

# 2 ZAB协议源码实现

先看下整体的实现情况，如下图所示

## 2.1 重要的数据介绍

LastProcessedZxid 和 maxCommittedLog minCommittedLog

## 2.1 Fast Leader Election

## 2.2 Recovery Phase

syncLimit 和 initLimit

## 2.3 Broadcast Phase



# 3 特殊情况的处理

## 3.1 重新选举leader时对之前数据的处理

### 3.1.1 重新选举前

造成重新选举有如下2种情况

-	Leader挂了

-	Leader得不到半数的follower支持


### 3.1.2 重新选举

raft对于entry是按照Index来进行处理的，一旦开启一个新的leader，新leader是需要处理前一个term的未提交的entry的
而zookeeper是按照epoch+index来进行处理的，一旦开启一个新的leader，新leader是不需要处理前一个term的未提交的entry的

## 3.2 follower挂了之后又重启的恢复过程

## 3.3 事务日志和快照日志的持久化

## 3.4 同步follower失败的情况

## 3.3 对client端是否一致

客户端收到OK回复，会不会丢失数据？
客户端没有收到OK回复，会不会多存储数据？






欢迎关注微信公众号：乒乓狂魔

![乒乓狂魔微信公众号](https://static.oschina.net/uploads/img/201610/28090041_LsUp.png "乒乓狂魔微信公众号")