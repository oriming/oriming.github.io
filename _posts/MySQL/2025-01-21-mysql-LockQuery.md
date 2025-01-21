---
title: MySQL 锁排查处理
author: Oriental Ming
date: 2025-01-21 18:13:00 +0800
categories: [MySQL, 锁]
tags: [MySQL]
render_with_liquid: false
---

# Welcome

本文主要做笔记，记录 MySQL 当出现锁表的时候的操作方式，怎么解锁:

**注意：MySQL8.1.0 作为演示环境。**

## 1. MySQL 事务

> 查询所需被锁的对象

```SQL
-- 查询出的结果，THREAD_ID 是 MySQL 内部线程ID
SELECT * FROM performance_schema.data_locks;
```

得到 MySQL 内部线程ID：`mysql_inner_thread_id`

## 2. 找到用户线程ID

```SQL
-- 依据 MySQL 内部线程ID 查询用户线程ID
SELECT * FROM performance_schema.threads WHERE THREAD_ID = <mysql_inner_thread_id>;
```

得到 `PROCESSLIST_ID`，这个就是我们要找的用户线程ID

## 3. Kill 它

```SQL
KILL <PROCESSLIST_ID>
```
