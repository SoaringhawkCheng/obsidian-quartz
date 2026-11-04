---
publish: true
title: 「大数据处理框架Apache Spark设计与实现 」读书笔记
created: 2026-10-18 11:27
modified: 2026-11-04T13:33:55.472Z
tags:
  - 
---

# 一、概览

## 1.1 系统架构

![[软件开发/数据引擎/大数据处理框架Apache Spark设计与实现/素材附件/Pasted image 20261022145908.png]]
如图为Spark集群的Standalone版本(资源管理和任务调度由Spark系统本身负责)，整个集群分为1个Master节点和3个Worker节点

- Master 节点上常驻 Master 守护进程，负责管理全部的 Worker 节点。
- Worker 节点上常驻 Worker 守护进程，负责与 Master 节点通信并管理 Spark任务的执行。
  启动集群后，用户可以在Master节点使用`./bin/run-example SparkPi 10`来提交一个应用
- Spark应用，一个可运行的Spark程序，包含main函数
- Spark驱动(Driver)，运行在Master节点的上Spark应用进程，负责执行main函数和创建SparkContext。如果是YARN集群，Driver也有可能被调度到Worker节点上运行
  Master节点收到应用后会通知Worker节点启动Executor
- Executor在物理上是一个JVM进程
- Driver在运行Spark应用的main函数时，会把应用拆成多个计算任务，Task以线程方式运行在Executor进程中。图中Worker节点1有8个CPU，启动了2个Executor，每个Executor可以并行运行4个task

## 1.2 应用例子

```
package org.apache.spark.examples

import java.util.Random

import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.SparkContext._

/**
  * Usage: GroupByTest [numMappers] [numKVPairs] [valSize] [numReducers]
  */
object GroupByTest {
  def main(args: Array[String]) {
    val sparkConf = new SparkConf().setAppName("GroupBy Test")
    var numMappers = 100 // mapper数量 = 并行分区数
    var numKVPairs = 10000 // 每个mapper生成多少条 KV，key是随机int
    var valSize = 1000 // 每个value是 1000 byte 的随机字节数组
    var numReducers = 36 // groupByKey之后reducer分区数量

    val sc = new SparkContext(sparkConf)

	// Size(arr1) = numKVPairs * (4 + valSize) = 10MB
	// Size(pairs1) = numMappers * Size(arr1) ＝1000MB
    val pairs1 = sc.parallelize(0 until numMappers, numMappers).flatMap { p =>
      val ranGen = new Random
      var arr1 = new Array[(Int, Array[Byte])](numKVPairs)
      for (i <- 0 until numKVPairs) {
        val byteArr = new Array[Byte](valSize)
        ranGen.nextBytes(byteArr)
        arr1(i) = (ranGen.nextInt(Int.MaxValue), byteArr)
      }
      arr1
    }.cache // 把这个 KV RDD 缓存到内存，后面重复计算不用重新生成
    // Enforce that everything has been calculated and in cache
    pairs1.count
    // 触发行动算子，强制把全部数据算出来、放进缓存
    // 如果不加这个 count，懒加载，等到 groupByKey 的时候才会生成数据，就测不准了。

    println(pairs1.groupByKey(numReducers).count)
    // 触发一次 **Shuffle**，按照 key 分组，分到 36 个 reducer 分区
    // 同一个 key 下所有 value，全部加载到 Reducer 的内存，封装成一个 Iterable
    // 统计一共有多少个不同的 key
    // 每个 reducer 收到的 <Int, Array[Byte]> record 个数为 `numMappers * numKVPairs / numReducer ＝ 27,777`，
    // 大小为 `Size(pairs1) / numReducer = 27MB`。

    sc.stop()
  }
}
```

阅读代码后，用户头脑中 job 的执行流程是这样的：
![[软件开发/数据引擎/大数据处理框架Apache Spark设计与实现/素材附件/Pasted image 20261022145549.png]]
使用`RDD.toDebugString`可以看到RDD的数据依赖关系如下
![[软件开发/数据引擎/大数据处理框架Apache Spark设计与实现/素材附件/Pasted image 20261022153906.png]]
针对这个 job，我们先画出它的物理执行 DAG 图如下：
![[软件开发/数据引擎/大数据处理框架Apache Spark设计与实现/素材附件/Pasted image 20261022155837.png]]可以看到 GroupByTest 这个 application 产生了两个 job

- 第一个 job 由第一个 action（也就是 `pairs1.count`）触发产生
  - 整个 job 只包含 1 个 stage，Stage 0 包含 100 个 ResultTask。
  - 每个 task 先计算 flatMap，产生 FlatMappedRDD，然后执行 action() 也就是 count()，统计每个 partition 里 records 的个数
  - pairs1 被声明要进行 cache，因此在 task 计算得到 FlatMappedRDD 后会将其包含的 partitions 都 cache 到 executor 的内存
  - task 执行完后，driver 收集每个 task 的执行结果，然后进行 sum()
- 第二个 job 由 `pairs1.groupByKey(numReducers).count` 触发产生
  - 整个 job 包含 2 个 stage。
  - Stage 1 包含 100 个 ShuffleMapTask，
  - Stage 0 包含 36 个 ResultTask，
  - task 执行完后，driver 收集每个 task 的执行结果，然后进行 sum()。

# 二、逻辑处理流程

![[软件开发/数据引擎/大数据处理框架Apache Spark设计与实现/素材附件/Pasted image 20261023102941.png]]

# 三、物理执行计划

**给定 job 的逻辑执行图，如何生成物理执行图（也就是 stages 和 tasks）？**
划分算法就是：**从后往前推算，遇到 ShuffleDependency 就断开，遇到 NarrowDependency 就将其加入该 stage。每个 stage 里面 task 的数目由该 stage 最后一个 RDD 中的 partition 个数决定。**
