---
layout: post
title: "Reids - 分布式锁 vs. zookeeper"
date: 2021-02-06 22:24:36 +0800
categories: Redis zookeeper lock
tags: Redis zookeeper lock
---

redis常被用来实现分布式锁。分布式锁和并发编程理念一致，唯一的区别就是前者是多个进程之间的协同，后者是同一个进程里多个线程之间的协同。

1. Table of Contents, ordered
{:toc}

# 分布式锁
逻辑上，分布式锁和非分布式锁是一样的：拿到锁的人干活，访问唯一资源，其他人不得干涉。

多个进程之间的分布式锁可以有很多种形式的实现，比如：一个进程创建一个文件，创建成功就算拿到锁了，可以访问临界资源，其他进程创建同名文件时发现文件已存在，创建失败，相当于没有拿到锁。

# Redis分布式锁
之所以那redis作为分布式锁，是因为redis用起来相当方便，部署启动方便，set/get也很方便。

在redis里，当一个key不存在时，set一个key/value成功了，就认为拿到锁了，处理完互斥操作后再把kv删除就行了，相当简单。当然，这个操作必须是原子的。redis提供了set if not exist的原子操作，即：
```
SETNX key value
```
正常情况下，一个分布式锁就这么实现了。一般这个value最好能唯一标志进程。

- setnx: https://redis.io/commands/setnx

## 进程锁 vs. 线程锁
但是进程间的分布式锁和线程之间的锁相比有个很大的不同点：持有锁的进程可能会崩掉，**或者更常见的情况，网络异常，导致节点掉线了**。其他进程还要继续工作，但是掉线进程设置的锁继续存在，其他进程将永远不能获得锁。

线程间的锁一般不需要考虑这些：
- 线程一般不会崩，基本是整个进程直接挂掉；
- 线程之间更不存在某个线程“网络异常导致掉线了”的情况；

像JVM这种线程间的锁不太需要考虑某个线程崩了怎么办，顶多就是进程崩了。既然进程都挂了，锁也不存在了。也不需要做什么后续处理。

所以redis分布式锁真正困难的地方是处理进程异常结束的特殊情况。

进程都崩了还怎么进行后处理操作把锁删了？一般能想到的就是给key设置一个过期时间。这就是基于一个假定：正常情况下一个操作用不了这么久，如果这么久key还在，可以理解为进程崩了、掉线了。key到时间自动过期。

这又衍生了两个问题：
- set if not exist本身需要原子操作，redis 的`setnx`指令可以做到，现在又要设定过期时间，这一步显然也要和上一步一起搞成原子操作。好在redis后来给`SET`指令进行了升级，支持set的同时设置过期时间：`SET key value time NX`，NX和setnx一样，代表not exist，即不存在时才能set；
- 万一特殊情况到时间了还没有执行完怎么办？key到时间就要自动删除了，其他进程岂不是也可以介入了？这种情况只能再开一个守护进程/线程，发现快到过期时间了还没结束，就再重设一下过期时间，“续一波”；

总之，用redis实现分布式锁，应对异常情况相当麻烦。

- set: https://redis.io/commands/set

# zookeeper分布式锁
redis之所以分布式锁实现的不够漂亮，就是因为**出现异常时，进程不在了，锁还在**。zk恰好能非常漂亮地解决这个问题，所以分布式锁一般用zk来搞。

## znode
zk为树状结构，就像一个文件系统。节点称为znode，有四种：
- persistent znode：持久节点，创建之后就存在，除非手动删了；
- persistent sequential znode：持久顺序节点，在同一个节点下创建节点时，zk给子节点自动编号，先创建的编号比后创建的小；
- ephemeral znode：临时节点，创建后存在，**client断开后，节点自动删除**；
- ephemeral sequential znode：临时顺序节点，在同一个节点下创建节点时，zk给子节点自动编号，先创建的编号比后创建的小，同时client断开后，节点自动删除；

**zk的分布式锁用的是临时顺序节点**，主要用了它的两个特性：
- ephemeral：client进程崩溃、掉线，节点就不再存在，**解决了redis在实现分布式锁时无法避免的痛**；
- sequential：相当于给每个进程编个号，我们可以定义从小到大为获取锁的顺序，则编号最小的节点对应的进程就是获取锁的进程；

zk获取锁的逻辑：
1. 创建一个parent znode；
2. 所有进程使用client在它下面创建“临时顺序”子znode；
3. 判断自己是不是编号最小的znode，是的话自己就获得了锁；
4. 不是的话，**只有等前一个znode消失了，才轮到自己，所以创建一个watch，监听前一个节点**。当前一个节点消失了，自己会收到watch的通知，现在自己就是最小的节点了，相当于获得了锁（当然，保险起见还是再检查一下自己是不是最小的）；

> **每一个节点等待前一个节点，形成一个等待队列**，AQS也这样：[AQS：显式锁的深层原理]({% post_url 2021-04-08-aqs %})

zk不同于redis，不是去“创建相同名称的node，创建成功为获得锁，不成功为没有获得锁”，而是创建顺序node，天然安排了一个获得锁的互斥顺序。所以**zk client进程不需要自旋等待锁**，只需要注册一个watch监听自己的前一个节点就行了。

# redis vs. zk
zk比redis优秀的地方：
- 进程异常时的处理：ephemeral，这点zk比redis好太多！所以用zk不需要考虑进程意外崩了掉线后的锁处理问题；
- **降低锁竞争**：**zk顺序节点已经安排了锁获取顺序，还提供了通知机制，每个节点静静等前一个节点的通知就行了**。redis client需要时不时尝试set kv，看能不能成功。没有通知，只能自己去不停重试了。也就是**自旋等待，实际上在竞争激烈时，这种机制会加剧竞争**。

当然，redis也有优点，比如set很快，部署也要简单很多。但总体来讲，在分布式锁这件事情上，还是zk更成熟更优秀。

# Ref
两篇很不错的文章：
- redis分布式锁：https://mp.weixin.qq.com/s/8fdBKAyHZrfHmSajXT_dnA
- zookeeper分布式锁：https://mp.weixin.qq.com/s/u8QDlrDj3Rl1YjY4TyKMCA
