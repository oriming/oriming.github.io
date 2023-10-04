---
title: MySQL 事务
author: Oriental Ming
date: 2023-10-04 11:13:00 +0800
categories: [MySQL, 事务]
tags: [MySQL]
render_with_liquid: false
---

# Welcome

本文主要做笔记，加深对 MySQL 事务的理解。内容主要有三点：

+ MySQL 事务隔离机制
+ 锁机制
+ MVCC(Multi-Version Concurrency Control) 多版本并发控制隔离机制

**注意：MySQL8.1.0 作为演示环境。**

## 1. MySQL 事务

> 数据库为了解决多事务并发问题，设计了**事务隔离机制、锁机制、MVCC隔离机制**, 用一整套机制来解决事务并发问题。

### 1.1 事务属性

事务是数据库系统中一系列操作的逻辑单元，所有操作要么全部成功要么全部失败！

事务是区分文件存储系统与 NoSQL 数据库的重要特性之一，其目的是为了保证即使在并发情况下也能正确的执行 CRUD 操作。

怎样去衡量结果的正确性？这时事务需要保证 ACID 特性：

+ 原子性（Atomicity）：事务是一个院子操作单元，对数据的修改，要么全部执行，要么全都不执行！
+ 一致性（Consistent）：在事务开始和完成时，数据都必须保持一致状态！
+ 隔离性（Isolation）：数据库系统提供一定的隔离机制，保证事务在不受外部并发操作影响的“独立”环境中执行！即事务处理过程中
  的中间状态对外部不可见，反之亦然！
+ 持久性（Durable）：事务完成后，它对数据的修改是永久性的，即使系统故障也能够保证！

### 1.2 事务隔离级别

如果想要在高并发场景下，完全保证 ACID 只需要把事务串行化执行即可，但是为什么在实际应用中不这样做呢，原因是因为性能会大大折扣！

在实际的业务场景中，往往不同的业务场景对事务的要求不一样，所以数据库设计了四种隔离级别，供我们选择：

**数据库默认隔离级别**

| DB | 隔离级别  |
| ----  | ----  |
| Oracle  | Read Committed  |
| MySQL  | Repeated Read  |

```mysql
-- 1. MySQL8 查询隔离级别
SELECT @@global.transaction_isolation, @@transaction_isolation;

-- 2.MySQL8 设置会话隔离级别

-- 读未提交
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- 读已提交
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- 可重复读
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- 串行化
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

```

### 1.3 并发事务问题

咱们先说结论：

| 隔离级别 | 脏读<br>Dirty Read | 不可重复读<br>Non-Repeatable Read | 幻读<br> Phantom Read |
| :----- | :----: | :----: | :----: |
| 读未提交（Read Uncommitted） | ✔️ | ✔️ | ✔️ |
| 读已提交（Read Committed） | ✖️ | ✔️ | ✔️ |
| 可重复读（Repeatable Read） | ✖️ | ✖️ | ✔️ |
| 串行化（Serializable） | ✖️ | ✖️ | ✖️ |

+ **脏读（Dirty Read）**

    简单讲就是，`事务A` 读取了 `事务B` 未 commit 的数据！违反了<u>**一致性**</u>要求!

    因为当 `事务B` Rollback后，`事务A` 之前读取就是脏数据，所以称为 **脏读**。

    **针对的是  <font color="red">Insert、Update、Delete</font>操作。**

    | 操作顺序 | 事务A | 事务B |
    | :-----: | :---- | :---- |
    | 1 | | Begin |
    | 2 | Begin | |
    | 3 | | 年龄从13岁更新为20岁 |
    | 4 | 查询是20岁（脏数据） | |
    | 5 | | Rollback，年龄 = 13岁 |
    | 6 | 业务操作增加5岁，更新为25岁 | |
    | 7 | commit | |

   结论：此业务逻辑，最终的岁数应该为 18岁，而不是 25岁!!!

+ **不可重复读（Non-Repeatable Read）**

    简单讲就是，`事务A` 相同的查询语句在不同时刻读出的结果不一致，违反了<u>**隔离性**</u>要求！

    **针对的是 <font color="red"> Update </font> 操作。**

    | 操作顺序 | 事务A | 事务B |
    | :-----: | :---- | :---- |
    | 1 | Begin | |
    | 2 | 首次查询，张三年龄20岁 | |
    | 3 |  | Begin |
    | 4 | 业务操作 | |
    | 5 | | 更新张三年龄为30岁 |
    | 6 | | commit |
    | 7 | 第二次查询，张三年龄变为30岁 | |

   结论：此业务逻辑，`事务A` 前后两次读取到的数据应该一致 !!!

   解决方案：使用行级锁，`事务A` 多次读取完成后才释放该锁，此时才允许其他事务更新这一行数据。

+ **幻读（Phantom Read）**

    简单讲就是，`事务A` 读取了 `事务B` commit 的数据，违反了<u>**隔离性**</u>要求！

    **针对的是 <font color="red"> Insert、Delete </font> 操作。**

    | 操作顺序 | 事务A | 事务B |
    | :-----: | :---- | :---- |
    | 1 | Begin | |
    | 2 | id=8数据不存在 | |
    | 3 |  | Begin |
    | 4 | 业务操作 | |
    | 5 | | 新增 id=8 |
    | 6 | | commit |
    | 7 | 新增id=8失败(主键冲突)，没有查询到 id=8 的数据 | |

   结论：`事务A` 明明没有查询到id=8的数据，但是提示主键冲突，好奇怪，跟出现了幻觉一样 !!!

   解决方案：使用表级锁，`事务A` 多次读取总量完成后才释放该锁，此时才允许其他事务新增数据。

## 2.锁机制

> 锁是计算机协调多个进程或线程并发访问某一资源的机制。
> 除了传统计算资源（CPU、RAM、I/O等）的争用之外，数据也是一种共享资源。如何保证数据并发访问的一致性、有效性是所有数据库
> 必须解决的一个问题！
> **锁冲突也是影响数据库并发访问性能的一个重要因素。**

### 2.1 锁分类

1. **性能上区分：** 乐观锁、悲观锁。
2. **数据库操作类型上区分：** 读锁、写锁。

    + **读锁：** 也称共享锁，S锁（**s**hared），针对同一份数据，多个读操作可以同时进行而不会相互影响。
    + **写锁：** 也称排它锁，X锁（e**x**clusive），当前的写操作未完成前，它会阻断其他写锁和读锁。

    | 当前\其他锁类型 | S锁 | X锁 |
    | :-----: | :----: | :----: |
    | S锁 | 兼容 | 互斥 |
    | X锁 | 互斥 | 互斥 |

    <u>不同语句不同的锁，如下：</u>
    | SQL | 锁类型 |
    | :----- | :----: |
    | INSERT、UPDATE、DELETE、<br/> SELECT...FOR UPDATE  | X锁 |
    | SELECT（通用） | 不加锁，快照锁 |
    | SELECT...FOR SHARE | S锁 |

3. **数据库操作粒度上区分：** 表锁、行锁。

    + **表锁：** 每次操作都锁住整张表。
        + 优点：开销小、加锁快；不会出现死锁。
        + 缺点：锁粒度大，锁冲突概率高，并发性能最低。
        + 适用场景：整表数据迁移。

    + **行锁：** MySQL InnoDB 引擎特有的特性。**并不是锁到记录上，而是加到了索引上！**
        + 优点：锁粒度小、并发性能高。
        + 缺点：死锁概率高。
        + 补充说明：按照锁的粒度划分，行级锁主要有记录锁（Record Lock）、间隙锁（Gap Locks）、临键锁（Next-Key Locks）。

            + 记录锁：宏观上看锁在了记录上，但实际锁在了索引上，当我们开启一个事务，使用`INSERT、UPDATE、DELETE、SELECT...FOR UPDATE` 语句
               操作某些已经存在的记录上的时候，就会加上记录锁；
            + 间隙锁：是一种范围锁，锁定的是一个区间（左开右开），**为了避免幻读的发生**。它的作用就是确保索引记录之间不能插入值（INSERT），避免产生幻读，
               在 RR（Repeatable Read） 事务隔离级别下支持。此锁因为并行度不够，冲突很多，所以容易引起死锁。
            + 临键锁：行锁和间隙锁的组合，同时锁住临界记录和间隙（左开右闭），在 RR（Repeatable Read） 事务隔离级别下支持。

## 3 WAL原则

> WAL，全称是Write-Ahead Logging， 预写日志系统。指的是 MySQL 的写操作并不是立刻更新到磁盘上，而是先记录在日志（日志成功写入后事务就不会丢失，后续由
> checkpoint 机制保证磁盘物理文件和 redo log 达到一致性）上，然后在合适的时间再更新到磁盘上。
> 这样的好处是错开高峰期。日志主要分为 `undo log`、`redo log`、`binary log`，作用如下：
>
> + `undo log` : 记录事务数据变更前的数据状态，用于回滚和其他事务多版本读。完成 MVCC 从而实现 MySQL 的隔离级别;
> + `redo log` : 记录事务变更后的数据状态。降低随机写的性能消耗（转成顺序写），同时防止写操作因为宕机而丢失;
> + `binary log` : 写操作的备份，保证主从一致;

## 4 MVCC(Multi-Version Concurrency Control)

> MySQL在可重复读隔离级别下如何保证事务较高的隔离性，同样的sql查询语句在一个事务里多次执行查询结果相同，就算其它事务对数据有修改也不会影响当前
> 事务 SQL 语句的查询结果。
> 这个隔离性就是靠MVCC机制来保证的，对一行数据的读和写两个操作默认是不会通过加锁互斥来保证隔离性，避免了频繁加锁互斥，而在串行化隔离级别为了保
> 证较高的隔离性是通过将所有操作加锁互斥来实现的。

**注意：MySQL 在 RC（Read Committed） 和 RR（Repeatable Read） 隔离级别下都实现了 MVCC 机制。**

### 4.1 实现

MVCC机制的实现就是通过 read-view 机制与 undo 版本链比对机制，使得不同的事务会根据数据版本链对比规则读取同一条数据在版本链上的不同版本数据。

### 4.2 undo日志版本链

对于使用InnoDB存储引擎的表来说，它的聚簇索引记录中都包含两个必要的隐藏列（row_id并不是必要的，我们创建的表中有主键或者非NULL唯一键时都不会包含row_id列）：

+ trx_id：每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列。
+ roll_pointer：每次对某条记录进行改动时，这个隐藏列会存一个指针，可以通过这个指针找到该记录修改前的信息。

### 4.3 read-view机制

对于使用RU（Read Uncommitted）隔离级别的事务来说，直接读取记录的最新版本就好了。对于使用 Serializable 隔离级别的事务来说，使用加锁的方式来访问记录。
对于使用RC（Read Committed）和 RR（Repeatable Read）隔离级别的事务来说，就需要用到我们上边所说的 `undo日志版本链`了，核心问题就是：需要判断一下版
本链中的哪个版本是当前事务可见的。

read-view中主要包含4个比较重要的内容：

+ m_ids：表示在生成ReadView时当前系统中活跃的读写事务的事务id列表。
+ min_trx_id：表示在生成ReadView时当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值。
+ max_trx_id：表示生成ReadView时系统中应该分配给下一个事务的id值。
+ creator_trx_id：表示生成该ReadView的事务的事务id。

> 注意max_trx_id并不是m_ids中的最大值，事务id是递增分配的。比方说现在有id为1，2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生
> 成ReadView时，m_ids就包括1和2，min_trx_id的值就是1，max_trx_id的值就是4。

有了这个ReadView，这样在访问某条记录时，只需要按照下边的步骤判断记录的某个版本是否可见：

+ 如果被访问版本的trx_id属性值与ReadView中的creator_trx_id值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。

+ 如果被访问版本的trx_id属性值小于ReadView中的min_trx_id值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问。

+ 如果被访问版本的trx_id属性值大于ReadView中的max_trx_id值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。

+ 如果被访问版本的trx_id属性值在ReadView的min_trx_id和max_trx_id之间，那就需要判断一下trx_id属性值是不是在m_ids列表中，如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。

------------

+ **RC 的实现方式：** 每次读取数据前都生成一个 read-view;
+ **RR 的实现方式：** 第一次读取数据前生成一个 read-view;

------------
<br/>
<br/>
<br/>
<br/>

部分内容摘自：

+ wego_xuchang <https://juejin.cn/post/6989900968833843231>
+ 超哥聊编程 <https://zhuanlan.zhihu.com/p/646142214>
+ 萌新J <https://www.cnblogs.com/mengxinJ/p/14211427.html>
