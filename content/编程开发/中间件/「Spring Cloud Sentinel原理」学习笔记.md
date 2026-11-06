---
publish: true
title: 「Spring Cloud Sentinel原理」学习笔记
created: 2023-05-01T12:09:00
modified: 2026-11-04T20:10:54.996Z
published: 2026-11-04T20:10:54.996Z
tags:
source:
  - https://www.jianshu.com/u/8c1316e385d7
---

# 零、Sentinel

相关文档

alibaba sentinel 官方中文文档 <https://sentinelguard.io/zh-cn/docs/introduction.html>

[Sentinel之集群流量控制介绍](https://www.jianshu.com/p/1227aa44c4ce)

[Sentinel之集群限流源码分析（一）](https://www.jianshu.com/p/17c642891261)

[Sentinel之集群限流源码分析（二）](https://www.jianshu.com/p/3537b292e5ee)

[Sentinel之Slots插槽源码分析（一）](https://www.jianshu.com/p/56e5ddaed640)

[Sentinel之Slots插槽源码分析（二）](https://www.jianshu.com/p/451548660fc1)

[Sentinel之Slots插槽源码分析系统规则（三）](https://www.jianshu.com/p/3668cc1bf24a)

[Sentinel之Slots插槽源码分析授权规则（四）](https://www.jianshu.com/p/ca186a5da2bb)

[Sentinel之Slots插槽源码分析流控规则（五）](https://www.jianshu.com/p/90531d69388f)

[Sentinel之Slots插槽源码分析降级规则（六）](https://www.jianshu.com/p/7eb1dcb67a6d)

[Sentinel之Slots插槽源码分析热点参数限流（七）](https://www.jianshu.com/p/533530e51aee)

[Sentinel核心源码解析](https://blog.csdn.net/qq_24101357/article/details/120653362)

[Sentinel 学习](https://blog.csdn.net/weixin_35821291/article/details/123836507)

![[编程开发/中间件/素材附件/Pasted image 20261105040320.png]]

# 一、启动和初始化

[Sentinel之从启动来分析加载过程](https://www.jianshu.com/p/dcc511e1ddbf)

## 1.1 自动装配

#### 1.1.1 spring.factories

![[编程开发/中间件/素材附件/Pasted image 20261105040328.png]]

#### 1.1.2 SentinelWebAutoConfiguration

![[编程开发/中间件/素材附件/Pasted image 20261105040335.png]]

#### 1.2 全局静态初始化

#### 1.2.1 Node初始化

###### 1.2.1.1 ROOT初始化

一个应用有一个ROOT节点，它是一个EntranceNode，由resource和ClusterNode构成

![[编程开发/中间件/素材附件/Pasted image 20261105040353.png]]

其中resource由name，entryType和resourceType唯一标识

![[编程开发/中间件/素材附件/Pasted image 20261105040359.png]]

![[编程开发/中间件/素材附件/Pasted image 20261105040402.png]]

###### 1.2.1.2 Context EntranceNode初始化

创建默认一个context，对应一个EntranceNode，这个EntranceNode同时是ROOT的子节点

![[编程开发/中间件/素材附件/Pasted image 20261105040412.png]]

![[编程开发/中间件/素材附件/Pasted image 20261105040417.png]]

###### 1.2.1.3 Tree结构

![[编程开发/中间件/素材附件/Pasted image 20261105040423.png]]

Node节点都有共同父类StatisticNode，其中各种Node的作用是：

1. StatisticNode：最为基础的统计节点，包含秒级和分钟级两个滑动窗口结构。
2. DefaultNode：链路节点，用于统计调用链路上某个资源的数据，维持树状结构。
3. ClusterNode：簇节点，用于统计每个资源全局的数据（不区分调用链路），以及存放该资源的按来源区分的调用数据。
4. EntranceNode：入口节点，特殊的链路节点，对应某个 Context 入口的所有调用数据。

![[编程开发/中间件/素材附件/Pasted image 20261105040434.png]]

#### 1.2.2 Env初始化

![[编程开发/中间件/素材附件/Pasted image 20261105040442.png]]

###### 1.2.2.1 Sph

sentinel process host，封装主流程操作

###### 1.2.2.2 InitExecutor

有一个initialized原子变量，保证初始化内容只会执行一次

这行代码就是通过JDK自带的SDK自带的SPI机制获取InitFunc的实现类了

![[编程开发/中间件/素材附件/Pasted image 20261105040508.png]]

# 二、主流程

## 2.1 Commonfilter

1. 调用ContexUtil.enter获取注册context
2. 调用SphU.entry执行子流程
3. 调用chain.doFilter调用下一个filter
4. 调用context的exit

![[编程开发/中间件/素材附件/Pasted image 20261105040515.png]]

![[编程开发/中间件/素材附件/Pasted image 20261105040521.png]]

## 2.2 contextUtil

#### 2.2.1 enter

从线程安全变量contextHolder中获取context

如不存在创建

![[编程开发/中间件/素材附件/Pasted image 20261105040533.png]]

#### 2.2.2 exit

![[编程开发/中间件/素材附件/Pasted image 20261105040540.png]]

## 2.3 SphU

#### 2.3.1 entry

1. 先获取或构建slotChain
2. 创建入口凭证Entry
3. 然后调用chain
4. 如遇到异常，调用Entry的exit方法

![[编程开发/中间件/素材附件/Pasted image 20261105040547.png]]

#### 2.3.2 lookProcessChain

一个resource对应唯一的chain，如果chainMap没有需要创建

![[编程开发/中间件/素材附件/Pasted image 20261105040556.png]]

SlotChainBuilder创建slotchain

![[编程开发/中间件/素材附件/Pasted image 20261105040602.png]]

![[编程开发/中间件/素材附件/Pasted image 20261105040609.png]]

## 2.4 Entry(入口凭证)

Entry是Sentinel中用来表示请求是否通过限流的一个凭证

每次执行 SphU.entry() 或 SphO.entry() 都会返回一个 Entry 给调用者。

资源调用结束时需要 entry.exit()

Entry包含了资源名、curNode（当前统计节点）、originNode（来源统计节点）等信息。

![[编程开发/中间件/素材附件/Pasted image 20261105040615.png]]

#### 2.4.1 Entry创建

创建entry并接入contex的entry链

![[编程开发/中间件/素材附件/Pasted image 20261105040620.png]]

#### 2.4.2 Entry退出

1. 错误的entry退出，清空context中的所有entry并抛出异常
2. 执行chain的exit
3. 从context的entry链表中移除当前entry

![[编程开发/中间件/素材附件/Pasted image 20261105040625.png]]

## 三、SlotChain

如2.1.3.3中述，一个resource拥有一个独立的chain

![[编程开发/中间件/素材附件/Pasted image 20261105040631.png]]

## 3.1 NodeSelectorSlot

[Sentinel之Entry构建源码解析](https://www.jianshu.com/p/3ae0733a3783)

#### 3.1.1 处理流程

一个resource可能被多个context调用，单例模式访问slot的map获取context对应的defaultNode

如果不存在则创建一个，并接入context的链表，调整context的curNode

![[编程开发/中间件/素材附件/Pasted image 20261105040637.png]]

#### 3.1.2 调用示例

1. 第一个例子ContextUtil.enter() 分别创建了前缀为entrance的两个上下文，同时指定调用发起者为appA

ContextUtil.enter("entrance1", "appA"); Entry nodeA = SphU.entry("nodeA"); if (nodeA != null) { nodeA.exit(); } ContextUtil.exit(); ContextUtil.enter("entrance2", "appA"); nodeA = SphU.entry("nodeA"); if (nodeA != null) { nodeA.exit(); } ContextUtil.exit();

树结构如下

machine-root / \ / \ entrance1 entrance2 -------表示入口节点对象EntranceNode / \ / \ DefaultNode(nodeA) DefaultNode(nodeB) ---------内部创建DefaultNode节点 | | | | ClusterNode(nodeA) ClusterNode(nodeB) ------------记录资源的访问数据

2. 第二个例子在同一个entrance上下文，通过entry请求两个不同的资源

ContextUtil.enter("entrance1", "appA"); Entry nodeA = SphU.entry("nodeA"); Entry nodeB = SphU.entry("nodeB"); if (nodeB != null) { nodeB.exit(); } if (nodeA != null) { nodeA.exit(); } ContextUtil.exit();

树结构如下

machine-root / / entrance1 -------表示入口节点对象EntranceNode / / DefaultNode(nodeA) ---------内部创建DefaultNode节点，持有一个ClusterNode对象 / / DefaultNode(nodeB) ------------记录资源的访问数据，持有一个ClusterNode对象

## 3.2 ClusterBuilderSlot

此插槽用于构建资源的 ClusterNode 以及调用来源节点。

ClusterNode 保持资源运行统计信息（响应时间、QPS、block 数目、线程数、异常数等）以及原始调用者统计信息列表。

![[编程开发/中间件/素材附件/Pasted image 20261105040647.png]]

## 3.3 StatisticSlot

用于记录，统计不同维度的runtime指标监控信息

![[编程开发/中间件/素材附件/Pasted image 20261105040651.png]]

#### 3.3.1 entry

1. 先调用fireEntry方法继续资源保护检查
2. 增加node的线程数increaseThreadNum和增加通过请求数addPassRequest
3. 如果originNode存在，则也需要增加originNode的线程数和请求通过数
4. 如果资源包装类型是IN的话，则需要增加ENTRY\_NODE的线程数和请求通过数
5. 调用StatisticSlotCallbackRegistry注册的callback，EntryMetricsCallback和ParamFlowStatisticCallback

![[编程开发/中间件/素材附件/Pasted image 20261105040657.png]]

#### 3.3.2 StatisticNode

包含了两个统计ArrayMetric，rollingCounterInSecond和rollingCounterInMinute

包含了一个curThreadNum

其中ArrayMetric是基于滑动窗口统计

![[编程开发/中间件/素材附件/Pasted image 20261105040704.png]]

###### 3.3.3 滑动窗口设计

[Sentinel之滑动时间窗口设计（一）](https://www.jianshu.com/p/824c886df5f6)

[Sentinel之滑动时间窗口设计（二）](https://www.jianshu.com/p/05677381e155)

###### 3.3.3.1 ArrayMetric 滑动窗口统计类

封装了实际数据LeapArray类型的data

![[编程开发/中间件/素材附件/Pasted image 20261105040711.png]]

并支持基于LeapArray的操作，具体见下一节

![[编程开发/中间件/素材附件/Pasted image 20261105040716.png]]

###### 3.3.3.2 LeapArray 窗口数组

windowLengthInMs：单口滑动时间窗口的大小，单位是毫秒

sampleCount：滑动时间窗口个数

intervalInMs：统计时间区间

array：保存统计数据的地方，是个AtomicReferenceArray

updateLock：更新窗口数据的时候用到

![[编程开发/中间件/素材附件/Pasted image 20261105040726.png]]

###### 3.3.3.3 LeapArray.currentWindow() 窗口定位

1. 计算当前time命中的窗口id，以及time对应windowStartTime

![[编程开发/中间件/素材附件/Pasted image 20261105040731.png]]

2. 根据idx获取窗口

3. 如果窗口不存在则创建，并通过cas加入到array

4. 如果存在

5. 窗口时间与计算的windowStartTime一致，立即返回

6. 不一致则需要更新窗口的start的时间，需要用到updateLock

![[编程开发/中间件/素材附件/Pasted image 20261105040739.png]]

![[编程开发/中间件/素材附件/Pasted image 20261105040758.png]]

###### 3.3.3.4 WindowWrap 窗口实现

![[编程开发/中间件/素材附件/Pasted image 20261105040830.png]]![[编程开发/中间件/素材附件/Pasted image 20261105040811.png]]

###### 3.3.3.5 MetricBucket 数据桶实现

包含了LongAdders数组counters，MetricEvent是枚举值，counters的size为MetricEvent的枚举数量

![[编程开发/中间件/素材附件/Pasted image 20261105040823.png]]

## 3.4 FlowSlot

[Sentinel之Slots插槽源码分析流控规则（五）](https://www.jianshu.com/p/90531d69388f)

#### 3.4.1 限流规则

![[编程开发/中间件/素材附件/Pasted image 20261105040842.png]]

###### 3.4.1.1 并发线程流量控制 FLOW\_GRADE\_THREAD

业内有使用隔离的方案：线程池隔离，信号量隔离

Sentinel线程数限流不负责创建和管理线程池，而是简单统计当前请求上下文的线程个数，如果超出阈值，新的请求会被立即拒绝

###### 3.4.1.2 QPS流量控制 FLOW\_GRADE\_QPS

![[编程开发/中间件/素材附件/Pasted image 20261105040849.png]]

通过selectNodeByRequesterAndStrategy方法选择被限流的节点

当 QPS 超过某个阈值的时候，则采取措施进行流量控制。流量控制的手段包括下面 4 种

1. 直接拒绝
2. 冷启动：让通过的流量缓慢增加，在一定时间内逐渐增加到阈值上限，给冷系统一个预热的时间，避免冷系统被压垮的情况

![[编程开发/中间件/素材附件/Pasted image 20261105040856.png]]

3. 匀速器：严格控制了请求通过的间隔时间，也即是让请求以均匀的速度通过，对应的是漏桶算法

![[编程开发/中间件/素材附件/Pasted image 20261105040902.png]]

4. 冷启动+匀速器

###### 3.4.1.3 基于调用关系的流量控制

![[编程开发/中间件/素材附件/Pasted image 20261105040908.png]]

1. 根据调用方限流：只根据某个入口的统计信息对资源限流
2. 关联流量控制：设置 FlowRule.strategy 为 RuleConstant.STRATEGY\_RELATE 同时设置 FlowRule.refResource
3. 链路限流：允许根据某个Entrance节点的统计信息对资源限流

#### 3.4.2 源码分析

## 3.5 ParamFlowSlot(实际在FlowSlot前面)

![[编程开发/中间件/素材附件/Pasted image 20261105040920.png]]

#### 3.5.1 ParamFlowChecker

## 3.6 DegradeSlot

[Sentinel之Slots插槽源码分析降级规则（六）](https://www.jianshu.com/p/7eb1dcb67a6d)
