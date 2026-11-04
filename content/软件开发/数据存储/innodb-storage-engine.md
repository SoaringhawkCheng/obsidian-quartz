---
publish: true
title: 「MySQL技术内幕·InnoDB存储引擎」学习笔记
created: 2019-06-15 21:09:15
modified: 2026-11-04T18:47:40.959Z
tags:
  - storage
---

# 体系结构

## 基础概念

- **数据库** 物理操作系统文件或其他形式文件类型的集合，是依照某种数据模型组织起来并存放于二级存储器的数据集合

- **实例** MySQL数据库由后台线程以及一个共享内存区组成。共享内存可以被运行的后台线程所共享，数据库实例才是真正用于操作数据库文件的。Mysql单进程多线程架构，实例就是一个进程

## 存储引擎

Mysql数据区别于其他数据库的最重要的一个特点就是其插件式的表存储引擎。存储引擎是基于表的，而不是数据库的。

数据处理系统分成两类：

- **联机事务处理OLTP**(Online Transaction Processing)，传统型关系数据库的主要应用，作为数据管理的主要手段，主要用作操作处理。
- **联机分析处理OLAP**(Online Analytical Processing)，数据仓库的最主要的应用，支持复杂的分析操作，侧重决策支持。

## SQL语言分类

SQL语言分四类：数据查询语言DQL，数据操纵语言DML，数据定义语言DDL，数据控制语言DCL

## 数据库连接

连接MySQL操作是一个连接进程和MySQL数据库实例进行通信

# 存储引擎

## InnoDB体系架构

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/innodb-storage-engine/1.png?raw=true)

- innodb内存池

主要作用有：

```
1. 维护所有进程/线程需要访问的多个内部数据结构
2. 缓存磁盘上的数据用于快速读取，缓存改动用于修改
3. 重做日志缓冲
```

- 后台线程

主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据

### 后台线程

- master线程

核心后台线程，负责将缓冲池中的数据异步刷新到磁盘

- io线程

innodb大量使用异步io来处理写io请求，io线程的作用主要是负责io请求的回调

- purge线程

事务提交后，undolog可能不再需要，需要purge线程回收已经使用并分配的undo页

- Page Cleaner线程

innodb1.2引入，将之前版本的脏页刷新操作放入到单独的线程中完成，目的是为了减轻master线程的工作和对用户查询线程的阻塞

### 内存

#### 缓冲池

由于CPU与磁盘速度之间的鸿沟，基于磁盘的数据库系统使用缓冲池技术来提高数据的整体性能

缓冲池中的数据类型有，默认的页的带下为16k：

```
索引页（主键索引非叶子，非主键索引）
数据页（主键索引叶子）
undo页
插入缓冲
自适应哈希索引
锁信息
数据字典信息
```

#### 内存管理

- LRU list

如果新读取到的页被放在LRU列表的首部，那么非常可能将热点数据页从LRU列表中移除，下次读取时需要再次访问磁盘

解决方案

1. **innodb\_old\_blocks\_pct** 数据库的缓冲池通过LRU算法进行管理，加入了midpoint位置，新读取到的页并不直接放入到LRU列表的首部，而是放入到LRU列表的midpoint位置，由innodb\_old\_blocks\_pct(通常3/8)控制

2. **innodb\_old\_blocks\_time** 用于表示页读取到mid位置后需要等待多久才能加入到LRU列表的热端，通过这个方法尽可能使LRU列表中热点数据不被刷出。

   **page made young**: 页从LRU列表的old部分加入到new部分

   **page not made young**: 因为block time的设置没有从old部分移到new部分

- free list

空闲页列表

- flush list

脏页列表。脏页既存在于LRU列表，也存在与Flush列表

#### 重做日志缓冲

首先将重做日志信息放入到缓冲区，然后按一定的频率将其刷新到外部磁盘的重做日志文件：

1. Master Thread每秒
2. 事务提交
3. 缓冲空间不到一半

### Checkpoint技术

页的操作是在缓存池完成的，当事务提交时，先做写重做日志，在修改页

Checkpoint记录脏页刷新回磁盘的情况，checkpoint前的页都已经刷新回磁盘

解决问题：

1. 缩短数据库恢复时间
2. 缓冲池不够用，将脏页刷新到磁盘
3. 重做日志不可用，刷新脏页

### MasterThread工作方式

### 关键特性

- 插入缓冲

非唯一辅助索引插入，如果不在缓冲区，先放入insert buffer

由于没有检查唯一性，所以不适用于唯一索引

- 两次写

具体见书，保证存储引擎数据页的可靠性

- 自适应哈希索引

Innodb会根据访问的频率和模式，为某些热点页建立哈希索引

- 异步IO

pipeline，写合并

- 刷新邻接页

刷新脏页时，检查同区的所有页，一同刷新

# 文件

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/innodb-storage-engine/2.png?raw=true)

## 日志文件

### 错误日志

### 慢查询日志

### 二进制日志

记录了对MySQL数据库执行更改的所有操作

主要作用：恢复，复制，审计

binlog格式：

1. statement 逻辑sql语句
2. row 记入表的更改情况
3. mixed 默认采用statement格式，部分情况采用row格式(见书)

## 套接字文件

UNIX域套接字

## pid文件

## 表结构定义文件

.frm文件，记录该表的表结构定义

## Innodb存储引擎文件

### 表空间文件

共享表空间 默认有个idbata1

独立表空间 表名.ibd文件

### 重做日志文件

重做日志缓冲往磁盘写入，按512字节一个扇区大小写，扇区是写入的最小单位，可以保证写入必定成功，不需要double write

与二进制日志区别：

1. 重做日志只记录事务日志，二进制日志记录所有与数据库相关的日志记录
2. 重做日志记录的是页的更改情况，是物理日志；二进制日志记录的是事务的操作内容，是逻辑日志
3. 重做日志在事务过程中，不断有重做日志条目写入重做日志缓冲，并按一定频率刷新到外部磁盘；二进制日志只在事务提交前提交，只写磁盘一次。

# 表

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/innodb-storage-engine/3.png?raw=true)

## 索引组织表

创建表没有显示定义主键，innodb会按如下方式选择或创建主键：

1. 是否有非空唯一索引
2. 创建6字节大小指针

## Innodb逻辑存储结构

### 表空间

共享表空间 回滚信息，插入缓冲索引页，事务信息，二次写缓冲

独立表空间 数据，索引和插入缓冲bitmap页

### 段

数据段 聚簇索引叶子节点

索引段 聚簇索引非叶子节点 二级索引

回滚段

### 区

1m，包含64个连续页

### 页

默认16k

## 分区

分区的过程是将一个表或者索引分解为多个更小，可管理的部分

mysql支持的分区类型：RANGE, LIST, HASH, KEY

- 分区和性能

OLTP 在线事务处理 分区会造成数倍IO

OLAP 在线分析处理 分区很好地提高查询性能

# 事务

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/innodb-storage-engine/4.png?raw=true)

## 认识事务

- 扁平事务

- 带有保存点的扁平事务

保存点记录事务的当前状态，以便后面发生错误，事务能回滚到保存点当时的状态

- 链事务

- 嵌套事务

任何事务只有在顶层事务提交才真正的提交，但是子事务支持回滚

- 分布式事务

## 事务实现

事务隔离性使用锁实现。原子性和持久性使用redo log实现，重做日志。undo log保证事务的一致性

### redo log

- 与binlog区别

1. bin log是mysql服务层产生，redo log是innodb存储器层产生
2. bin log是逻辑日志，对应的sql语句，redo log是物理日志，记录的每个页的修改
3. bin log在事务提交完成后进行一次写入，redo log在事务进行，不断有重做日志条目写入重做日志缓冲，并按一定频率刷新到外部磁盘

- 重做日志块

重做日志缓存，重做日志文件都是以块的方式保存，重做日志块大小512字节，和磁盘扇区大小一样，重做日志写入保证原子性，不需要double write技术

- LSN

表示事务写入重做日志字节的总量

- 幂等性

重做日志是幂等的，二进制日志不是

- 生成与删除

事务开始之后就产生redo log，事务脏页写入磁盘后重用

### bin log

- 生成与删除

事务提交的时候，一次性将事务中的sql语句计入binlog中

具体binlog提交见内部XA事务

### undo log

undo是逻辑日志，只是将数据库逻辑地恢复到原来的样子，主要有两个功能

1. 用于回滚

2. MVCC 当前事务可以通过undo读取之前的行版本信息，以此实现非锁定读取

undo log分为insert和update undo log

update log记录的是对delete和update操作产生的undo log，用于MVCC机制，事务提交时就进行删除，提交时放入undo log链表，等待purge线程删除

- 生成与删除

事务开始之前，将当前是的版本生成undo log，undo也会产生 redo 来保证undo log的可靠性

事务提交时，放入undo log链表，若没有事务引用，等待purge线程删除

### purge

### group commit(不懂，冷门，跳过)

## 分布式事务

### XA事务

XA事务由一个或多个资源管理器，一个事务管理器以及一个应用程序组成

资源管理器 Resource Managers

事务管理器 Trasaction Manager

应用程序 Application Program

### 内部XA事务

QPS 每秒请求数

TPS 每秒处理事务数

### 事务Rollback与崩溃恢复(见软件架构设计)

# 索引与算法

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/innodb-storage-engine/5.png?raw=true)

## B+树索引

### B+树索引

- 聚集索引

B+Tree的高度一般都在2~4层。mysql的InnoDB存储引擎在设计时是将根节点常驻内存的，也就是说查找某一键值的行记录时最多只需要1~3次磁盘I/O操作

数据页存放的是完整的每行的记录

非数据页存放的是键值和指向数据页的偏移量

- 辅助索引

### 全文检索

# 锁

![](https://github.com/SoaringhawkCheng/blog/blob/master/source/_posts/innodb-storage-engine/6.png?raw=true)

## 锁的类型

### 锁的类型

- 行级锁

Innodb实现了两种行级锁：

共享锁 读一行数据

排他锁 删除或更新一行数据

- 意向锁

为了允许事务在行级上和表级上的锁同时存在，支持不同粒度加锁操作。Innodb支持意向锁

意向共享锁 事务想要获得表中某几行数据的共享锁

意向排他锁 事务想要获得表中某几行数据的共享锁

- 兼容性

IS/IX的笛卡尔积相互兼容,S/IS的笛卡尔积相互兼容

### 一致性非锁定读

READ COMMITTED 读取被锁定行的最新一份快照数据

REPEATABLE RAED 总是读取事务开始的版本行数据

事务的隔离级别为REPEATABLE READ，Innodb存储引擎的SELECT操作使用一致性非锁定读

### 一致性锁定读

- SELECT ... FOR UPDATE

对读取的行记录加一个X锁，该行不能再加任何锁

- SELECT ... LOCK IN SHARE MODE

对读取的记录加一个S锁，其他事务可以对该行加S锁，加X锁会被阻塞

事务提交锁就释放了

### 自增长锁

每个含有自增长值的表，都有一个自增长计数器，对于含有自增长计数器的表进行插入，加表锁AUTO-INC Locking

自增长的值，必须是索引的第一列

## 锁的算法

- Record Lock:

单个行记录上的锁

- Gap Lock: 间隙锁

锁定一个范围，不包括记录本身

阻止多个事务将记录插入到统一范围

- Next-Key Lock:

Gap Lock+Record Lock，

锁定一个范围，并且锁定记录本身

## 阻塞

因为不同锁之间的兼容性，一个事务的锁需要等待另一个事务中的锁释放它锁占用的资源

## 死锁

事务因争夺锁资源造成互相等待的现象，采用等待图进行死锁检测
