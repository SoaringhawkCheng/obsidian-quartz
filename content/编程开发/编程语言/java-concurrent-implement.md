---
publish: true
title: 「Java并发实现原理：JDK源码剖析」学习笔记
created: 2021-11-12 15:51:25
modified: 2026-11-06T07:06:39.545Z
published: 2026-11-06T07:06:39.545Z
tags:
source:
  - https://book.douban.com/subject/35013531/
---

# 一、线程基础

## 1.1 线程状态机

![[编程开发/编程语言/素材附件/20261106-150639-001.png]]

[笔记](https://www.jianshu.com/p/74295daa17c3)

[状态转换](https://blog.csdn.net/sufu1065/article/details/120170509)

[2](https://blog.csdn.net/weixin_41489022/article/details/113109184)

## 1.2 线程中止

[笔记](https://www.jianshu.com/p/b5e0188f80a8)

stop、destroy之类的函数不建议使用，强制杀死线程，线程中所使用的资源如描述符和网络连接无法正常关闭，通过传递标志位来完成线程退出操作

JVM中线程分为守护线程和非守护线程，所有非守护线程退出后，整个JVM进程就会退出

## 1.3 线程通信

[笔记](https://www.jianshu.com/p/e54ab941a147)

## synchronized

## wait与notify

## 1.4 线程封闭

[笔记](https://www.jianshu.com/p/8d305f178685)

## interrupt

## volatile关键字

# Java内存模型

[笔记](https://www.jianshu.com/p/337ad6f11a6b)

JVM为了跨平台，实现了Java内存模型用来屏蔽掉各种硬件和操作系统的内存访问差异

JMM的目标是定义程序中各个变量的访问规则，即在JVM中将变量从内存中取出和存入的底层细节

JMM的关键技术点都是围绕多线程的原子性、可见性和有序性来建立的

## 三个概念

### 原子性

对于32位系统来说，long型数据的读写不是原子性的

### 可见性

#### 缓存一致性问题

![[编程开发/编程语言/素材附件/20261106-150639-002.webp]]

如果一个变量在多个CPU中都存在缓存（一般在多线程编程时才会出现），那么就可能存在缓存不一致的问题

为了解决缓存不一致性问题，通常来说有以下2种解决方法：

- 通过在总线加LOCK锁，在锁住总线期间，其他CPU无法访问内存，导致效率低下
- 通过缓存一致性协议

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/java-concurrent-implement/cpu-cache-consistency.jpeg)

#### 缓存一致性协议

缓存同步协议MESI，规定每条缓存有个状态位，定义了四种状态M、E、S和I

单个CPU对缓存中数据进行了改动，需要通知给其他CPU；这意味着CPU不仅要控制自己的读写操作

缓存一致性协议可以保证CPU缓存一致，但是对性能有很大的消耗，因此CPU的架构师在计算单元和L1之前又增加了 Store Buffer、Load Buffer。也就是说，往内存中写入一个变量，这个变量会保存在 Store Buffer 里面，稍后才异步写入L1中，同时同步写入主内存中

![[编程开发/编程语言/素材附件/20261106-150639-003.webp]]

### 有序性

![[编程开发/编程语言/素材附件/20261106-150639-004.webp]]

#### 重排序

重排序分类：

- 编译器重排序：对于没有先后依赖关系的语句，编译器可以重新调整语句的执行顺序
- CPU指令重排序：当CPU写缓存时发现缓存区块正在被其他CPU占用，为了提高CPU处理性能，可能将后面的读缓存命令优先执行
- CPU内存重排序：读写都是在 Load Buffer 和 Store Buffer 中完成的，缓存中的数据不会马上同步到主内存中去，因此数据同步到主内存中的顺序可能跟指令执行的顺序不一致，如下图所示

![[编程开发/编程语言/素材附件/20261106-150639-005.webp]]

#### as-if-serial

编译器和CPU只能保证每个线程的as-if-serial语义，即单线程程序的执行结果不能改变

对于多线程场景，需要上层告知编译器和CPU是否可以重排序

#### happen-before

- 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作
- 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作
- volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作
- 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C
- 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
- 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束
- Thread.isAlive()的返回值手段检测到线程已经终止执行
- 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始

## 解决方案

## 内存屏障

为了禁止编译器重排序和CPU重排序，在编译器和CPU层面都有内存屏障指令，也是JMM和happen-before规则的底层实现原理

### Linux内核例子

kfifo.c源代码中RingBuffer

在修改数据和更新指针之间，通过smp\_wmb()插入了一个Store Barrier，从而确保两个操作：

- 更新指针操作不会被重排到修改数据前
- 数据的修改在更新指针前，已经完成Store Cache刷新，其他CPU可见

### JDK内存屏障

读内存屏障loadfence：在指令前插入Load Barrier，可以让高速缓存中的数据失效，强制重新从主内存加载数据，让CPU缓存与主内存保持一致，避免缓存导致的一致性问题。

写内存屏障writefence：在指令后插入Store Barrier，能让写入缓存中的最新数据更新写入主内存，让其他线程可见；当发生这种强制写入主内存的显式调用，CPU就不会处于性能优化考虑进行指令重排。

fullfence：loadfence + writefence

![[编程开发/编程语言/素材附件/20261106-150639-006.webp]]

### volatile

#### 功能

- 64位写入的原子性：32位虚拟机下，long型变量的原子写入
- 内存可见性：多线程数据强一致性，即写完之后会立即对其他线程可见
- 禁止重排序：

#### 读写规则

对一个volatile变量的单个读/写操作，与对一个普通变量的读/写操作使用同一个锁来同步，执行效果是相同的

- 当写一个volatile时，JMM会把该线程对应的本地内存中的共享变量刷新到内存。
- 当读一个volatile时，JMM会把该线程对应的本地内存中的共享变量置为无效，线程接下来将从主内存中读取共享变量

# SDK并发包

![[编程开发/编程语言/素材附件/20261106-150639-007.webp]]

## Atomic类

### 线程安全机制

- 等待唤醒机制：当长时间都无法抢到锁的时候，还是将线程挂起，然后等待唤醒的好。因为等待和唤醒牵扯到线程挂起和切换，会导致从用户态到内核态的切换，并且线程切换会导致上下文的切换，现场保存什么的，会比较浪费资源
- 自旋CAS：当短时间内就可以获取到锁的时候，自旋CAS比较合适，短时间的自旋CAS肯定会比线程切换消耗的资源要少，如果要是时间长的话，就不太划算了，因为自旋CAS会一直占用CPU

### atomic类型

unsafe提供了三种类型的cas操作：int，long，object

对应的基本类型有AtomicInteger、AtomicBoolean、AtomicLong、AtomicReference和Atomic{type}Array

AtomicStampedReference通过引入时间戳来解决了ABA问题。每次要更新值的时候，需要额外传入oldStamp和newStamp，将对象和stamp包装成了一个Pair对象

Atomic{type}Updater用来修改实例对象中的属性的值的，被修改变量必须用volatile修饰

### 高并发原子累加器striped64

![[编程开发/编程语言/素材附件/20261106-150639-008.png]]

![[编程开发/编程语言/素材附件/20261106-150639-009.png]]

多线程对变量进行CAS，高并发场景下仍不够快，将long拆成base和cells，求和的时候并不加锁，只能保证最终一致性

{type}Addr进行累加，{type}Accumulator可以定义二元操作符

## Lock和Condition

### 基本原理

AbstractQueuedSynchronizer是用来构建锁或者其他同步组件的基础框架

#### 核心要素

锁有以下核心要素：

- 保存一个state变量标记该锁的状态，标识持有线程数量，对state变量的操作使用CAS保证线程安全
- 记录当前持有锁的线程
- 基于底层的park()和unpark()阻塞自己，或者唤醒线程
- 使用一个线程安全的无锁队列维护所有的阻塞的线程

#### 阻塞队列实现

双向链表，有head和tail指针

入队流程：

```
pred = tail // 获取tail的快照pred
node.prev = pred // 新节点node的prev指向pred
if (CAS(pred, node)) { // 如果tail==pred，将tail设置为pred
	pred.next=node
}
```

出队流程：release中调用unparkProcessor，由于release会判断是否持有锁，所以不需要CAS

### 锁操作分类

公平性和非公平性：公平锁先排队，非公平锁先cas锁的state，默认为非公平锁

可中断锁和不可中断锁：不可中断锁会记录阻塞期间是否有中断信号，锁唤醒后返回中断状态处理；可中断锁，阻塞时收到信号会直接抛出异常

tryLock：非公平锁的一种，但是cas锁的state，如果失败就会返回不会轮询

读写锁：将state分为高低位，分别保存写锁和读锁state

### Condition

![[编程开发/编程语言/素材附件/20261106-150639-010.png]]

![[编程开发/编程语言/素材附件/20261106-150639-011.png]]

从队列（同步队列和等待队列）的角度看await()方法，当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中

![[编程开发/编程语言/素材附件/20261106-150639-012.png]]

调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中

## 同步工具类

[图解CyclicBarrier](https://www.cnblogs.com/crazymakercircle/p/13906379.html)

[图解Semaphore](https://www.cnblogs.com/crazymakercircle/p/13907012.html)

[图解CountDownLatch](https://www.cnblogs.com/crazymakercircle/p/13906922.html)

## 并发容器

### BlockingQueue

ArrayBlockingQueue：数组实现的环形队列，一个lock两个condition

LinkedBlockingQueue：单向链表，FIFO，两个lock和两个condition

PriorityBlockingQueue：底层最小二叉堆，并实现compare接口

DelayQueue：底层是PriorityQueue（非线程安全），一个lock一个condition，时间堆

SynchronousQueue：基于TransferQueue（公平模式）和TransferStack（非公平），底层是单向链表

LinkedBlockingDeque：双向链表，支持双端队列操作接口

### ConcurrentHashMap

使用红黑树，加锁粒度是头结点，并发扩容

[扩容机制](https://blog.csdn.net/Mind_programmonkey/article/details/111035223)

### ConcurrentSkipListMap

# 线程池与Future

# ForkJoinPool

# CompleteFuture
