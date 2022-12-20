---
title: Linux 文件处理的技巧
author: Sunny Boy
date: 2022-12-20 21:04:00 +0800
categories: [Linux, 运维]
tags: [Linux, 运维]
render_with_liquid: false
---

> 收集整理出一些在运维时常用的文件处理技巧。后续会对内容再补充完善。

## 清理 nohup.out

> 生产中因为长时间不处理nohup.out文件，导致文件会特别的大，在查询日志等方面会非常的不方便。在
> 清理nohup.out文件时保持程序的正常运行就显得比较重要了。
>
> ⚠️ 注意：如果应用本身有日志记录策略，则在启动时完全可以不输出 nohup.out 文件 😁

```bash
# 清理nohup.out文件至空
cp /dev/null nohup.out
```

**启动时不输出 nohup.out，已 jar 包启动为例：**

```bash
# 说明：将 nohup 命令的标准输出日志输出到 /dev/null 内（也就是垃圾桶）
nohup java -jar nectarine.jar &> /dev/null &
```

-----

## 清除 Windows 系统回车符

> 找到两种解决方式，一种是 `vim` ，另一种是软件 `dos2unix`，两者都好用不复杂！😇

1. vim 方式

    + **打开文件**

        ```bash
        # 打开指定的文件
        vim filename
        # 切到命令行模式(按下英文冒号:)。设置让它显示回车符
        e ++ff=unix %
        ```

    + **清除**

        ① 替换的方式

        ```bash
        # 首先按下英文冒号（关注左下角）
        # 借助替换命令将回车符去掉
        # ^M的输入方法先按按键盘 Ctrl + V ➜ ^，再按 Ctrl + M ➜ M
        %s/^M//g
        # 保存
        wq
        ```

        ② 另一种的方式

        ```bash
        # 首先按下英文冒号（关注左下角）
        # 设置文件的文件格式
        set fileformat=unix
        # 保存
        wq
        ```

2. dos2unix

> 是dos文件转换(to)成unix格式文件，格式化。

```bash
# 如果没有 dos2unix 命令，需要先安装
dos2unix fileName
```
