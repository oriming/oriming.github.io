---
title: Linux 查询技巧
author: Oriental Ming
date: 2022-12-20 21:04:00 +0800
categories: [Linux, 查询技巧]
tags: [Linux]
render_with_liquid: false
---

> 收集整理出一些在运维时常用的查看技巧。后续会对内容再补充完善。

## 查本机外网 IP

```bash

# 可能需要等待 2秒钟左右 😜
curl ifconfig.co
```

-----

## 查 PID 文件启动位置

```bash
# 先查 PID。例：某个正在启动的 jar 包 PID
ps aux | grep java

# 获取到 PID 之后（假设：188），查看文件位置

ll /proc/188 | grep cwd
```

-----

## 查文件数量

```bash
# 统计当前目录下文件的个数（不包括目录）
ls -l | grep "^-" | wc -l

# 统计当前目录下文件的个数（包括子目录）
ls -lR| grep "^-" | wc -l

# 查看某目录下文件夹(目录)的个数（包括子目录）
ls -lR | grep "^d" | wc -l
```

-----

## 查文件大小

```bash
# 显示此目录下所有文件的总大小
du -sh
21G

# 查询目录下前10大小的文件
du -sh * | sort -nr | head

# 查看目录下大小超过100M的文件
find . -type f -size +100M
```

-----

## 查 IO 性能（粗略）

> 依托自身的dd命令 粗略查看磁盘IO性能。测试前需要先把
> 缓存清理之后再测试。

1. `root` 权限清缓

    ```shell
    # 清除缓存
    echo 3 > /proc/sys/vm/drop_caches
    ```

2. 写的性能

    ```shell
    # offlag=direc 参数测试 IO 时必须指定，代表直接写如磁盘，不使用 cache
    dd if=/dev/zero of=sun.dd bs=1M count=200 oflag=direct
    # 输出如下
    200+0 records in
    200+0 records out
    209715200 bytes (210 MB) copied, 4.57099 s, 45.9 MB/s
    ```

3. 读的性能

    ```shell
    # ifflag=direc 参数测试 IO 时必须指定，代表直接读磁盘，不使用 cache
    dd of=/dev/zero if=sun.dd bs=1M count=200 iflag=direct
    # 输出如下
    200+0 records in
    200+0 records out
    209715200 bytes (210 MB) copied, 1.01719 s, 206 MB/s
    ```

-----

## 查 history

> 有的时候我们想查看那个IP操作过哪些内容，可以通过history命令实现。

```bash
# 设置history展示的格式  时间 ip
export HISTTIMEFORMAT="%F %T `who am i` "

# 使用grep、less搜索即可(例子：)
history | grep redis | less

展示效果：

  257  2021-09-28 12:55:01 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) cd /usr/bin/redis*
  258  2021-09-28 12:55:01 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) ls /usr/libexec/redis*
  259  2021-09-28 12:55:01 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) cd /usr/local/redis/
  732  2021-09-28 12:55:01 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) cd /usr/local/redis/
  736  2021-09-28 12:55:01 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) ./redis-cli
  990  2021-09-28 12:45:09 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) ps aux | grep redis
  991  2021-09-28 12:45:20 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) history | grep redis | less
  993  2021-09-28 12:45:54 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) history -c | grep redis | less
  997  2021-09-28 12:48:30 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) history | grep redis | less
  999  2021-09-28 12:50:47 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) history | grep redis | less
 1001  2021-09-28 12:55:28 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) history | grep redis | less
 1003  2021-09-28 12:55:41 zhangsan pts/1        2021-09-28 12:55 (122.5.45.202) history | grep redis | less
```

-----

## locate 替代 find

> 使用locate的理由:  使用find会占用系统的大量资源，在生产上使用find命令是禁忌。
> 而locate会建立一个文件资料库， 节省很多系统的资源。

**⚠️ 注意：此命令会略过 /tmp 目录下的文件！**

```bash
# 使用前, 更新文件库
updatedb
# 使用
locate filename
```
