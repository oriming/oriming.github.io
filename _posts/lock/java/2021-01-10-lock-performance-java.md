---
title: 锁(Java)性能探究
author: Oriental ming
date: 2023-05-06 16:14:00 +0800
categories: [锁, 性能]
tags: [Java, Lock, 性能]
render_with_liquid: false
---

## 前言

环境 | 备注
------ | ---
JDK | 版本 8
OS |  macOS Ventura 13.1
CPU | M1 Pro
内存 | 32G
磁盘 | APPLE SSD AP0512R

## 测试的锁类型

锁方式 | 对应方法
----|---
synchronized  方法锁|  syncByMethod
synchronized 常量锁|  syncByConstant
synchronized 对象锁  | syncByThis
StampedLock 写锁 | stampedLock
ReentrantLock | reentrantLock

## 结果

```java
  // syncByMethod 方式, 共执行 20组，每组 100次，最长耗时 2540毫秒, 最短耗时 128毫秒, 平均耗时 1414.950000毫秒
  // syncByMethod 方式, 共执行 20组，每组 1000次，最长耗时 25461毫秒, 最短耗时 3702毫秒, 平均耗时 19282.150000毫秒

  // syncByThis 方式, 共执行 20组，每组 100次，最长耗时 2564毫秒, 最短耗时 127毫秒, 平均耗时 1542.350000毫秒
  // syncByThis 方式, 共执行 20组，每组 1000次，最长耗时 25560毫秒, 最短耗时 3598毫秒, 平均耗时 19210.350000毫秒

  // syncByConstant 方式, 共执行 20组，每组 100次，最长耗时 2541毫秒, 最短耗时 128毫秒, 平均耗时 1485.900000毫秒
  // syncByConstant 方式, 共执行 20组，每组 1000次，最长耗时 25605毫秒, 最短耗时 14804毫秒, 平均耗时 21494.650000毫秒

  // stampedLock 方式, 共执行 20组，每组 100次，最长耗时 2541毫秒, 最短耗时 2419毫秒, 平均耗时 2497.800000毫秒
  // stampedLock 方式, 共执行 20组，每组 1000次，最长耗时 25272毫秒, 最短耗时 25085毫秒, 平均耗时 25215.000000毫秒

  // reentrantLock 方式, 共执行 20组，每组 100次，最长耗时 2550毫秒, 最短耗时 128毫秒, 平均耗时 1479.400000毫秒
  // reentrantLock 方式, 共执行 20组，每组 1000次，最长耗时 25592毫秒, 最短耗时 1362毫秒, 平均耗时 16253.050000毫秒
```

**总结：**
在普通场景下使用，各种锁的性能相差无几。

## 代码

### 1.源代码

```java
package priv.utrix.explore8.demo;

import lombok.Getter;
import lombok.Setter;
import lombok.SneakyThrows;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.StampedLock;

/**
 * 锁的性能测试
 * <pre>
 *    总结：synchronized经过不断的优化，到目前JDK8版本已经不再是低级笨重的锁了，
 *    在一起其他场合性能已经优于reentrantLock和stampedLock了
 *
 *    原因：synchronized引入了偏向锁、轻量级锁、重量级锁，并提供了自动的升级降级机制
 * </pre>
 *
 * @author Oriental Ming
 * @date 2023/5/6 16:35
 */
public class LockPerformanceDemo {


    /**
     * 模拟争抢的资源
     */
    @Getter
    private long resources = 0;
    private static final Object LOCK_KEY = new Object();

    @Setter
    private StampedLock lock;
    @Setter
    private ReentrantLock reentrantLock;

    /**
     * synchronized 方法锁
     * 方法锁锁定的该方法所属对象隐藏字段的monitor
     */
    @SneakyThrows
    public synchronized void syncByMethod() {
        TimeUnit.MILLISECONDS.sleep(1);
        resources++;
    }

    /**
     * synchronized this 对象锁
     */
    @SneakyThrows
    public void syncByThis() {
        synchronized (this) {
            TimeUnit.MILLISECONDS.sleep(1);
            resources++;
        }
    }

    /**
     * synchronized 对象锁，锁内容
     */
    @SneakyThrows
    public void syncByConstant() {
        synchronized (LOCK_KEY) {
            TimeUnit.MILLISECONDS.sleep(1);
            resources++;
        }
    }

    /**
     * JDK8 StampedLock锁
     */
    @SneakyThrows
    public void stampedLock() {
        Lock writeLock = lock.asWriteLock();
        try {
            writeLock.lock();

            TimeUnit.MILLISECONDS.sleep(1);
            resources++;
        } finally {
            writeLock.unlock();
        }
    }

    /**
     * ReentrantLock 锁
     */
    @SneakyThrows
    public void reentrantLock() {
        try {
            reentrantLock.lock();

            TimeUnit.MILLISECONDS.sleep(1);
            resources++;
        } finally {
            reentrantLock.unlock();
        }
    }

}
```

### 2.测试源代码

```java
package priv.utrix.explore8.demo;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.date.TimeInterval;
import cn.hutool.core.thread.ThreadUtil;
import com.google.common.collect.Lists;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import java.util.List;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.StampedLock;
import java.util.stream.Collectors;

import static org.junit.jupiter.api.Assertions.assertEquals;

class LockPerformanceDemoTest {

    private LockPerformanceDemo service;
    /**
     * 运行多少组
     */
    public int groupNumber;
    /**
     * 每组运行的次数
     */
    public int perGroupNumber;
    public List<Long> timeCountList;

    @BeforeEach
    void setUp() {
        service = new LockPerformanceDemo();
    }

    @AfterEach
    void afterEach() {
        assertEquals((long) groupNumber * perGroupNumber, service.getResources());
    }

    @ParameterizedTest
    @CsvSource({"20, 100", "20, 1000"})
    void syncByMethod(int groupCount, int perGroupCount) {
        groupNumber = groupCount;
        perGroupNumber = perGroupCount;
        timeCountList = Lists.newArrayListWithCapacity(groupNumber * perGroupCount);

        reduce("syncByMethod", service::syncByMethod);
        // syncByMethod 方式, 共执行 20组，每组 100次，最长耗时 2540毫秒, 最短耗时 128毫秒, 平均耗时 1414.950000毫秒
        // syncByMethod 方式, 共执行 20组，每组 1000次，最长耗时 25461毫秒, 最短耗时 3702毫秒, 平均耗时 19282.150000毫秒
    }

    @ParameterizedTest
    @CsvSource({"20, 100", "20, 1000"})
    void syncByThis(int groupCount, int perGroupCount) {
        groupNumber = groupCount;
        perGroupNumber = perGroupCount;
        timeCountList = Lists.newArrayListWithCapacity(groupNumber * perGroupCount);

        reduce("syncByThis", service::syncByThis);
        // syncByThis 方式, 共执行 20组，每组 100次，最长耗时 2564毫秒, 最短耗时 127毫秒, 平均耗时 1542.350000毫秒
        // syncByThis 方式, 共执行 20组，每组 1000次，最长耗时 25560毫秒, 最短耗时 3598毫秒, 平均耗时 19210.350000毫秒
    }

    @ParameterizedTest
    @CsvSource({"20, 100", "20, 1000"})
    void syncByConstant(int groupCount, int perGroupCount) {
        groupNumber = groupCount;
        perGroupNumber = perGroupCount;
        timeCountList = Lists.newArrayListWithCapacity(groupNumber * perGroupCount);

        reduce("syncByConstant", service::syncByConstant);
        // syncByConstant 方式, 共执行 20组，每组 100次，最长耗时 2541毫秒, 最短耗时 128毫秒, 平均耗时 1485.900000毫秒
        // syncByConstant 方式, 共执行 20组，每组 1000次，最长耗时 25605毫秒, 最短耗时 14804毫秒, 平均耗时 21494.650000毫秒
    }

    @ParameterizedTest
    @CsvSource({"20, 100", "20, 1000"})
    void stampedLock(int groupCount, int perGroupCount) {
        groupNumber = groupCount;
        perGroupNumber = perGroupCount;
        timeCountList = Lists.newArrayListWithCapacity(groupNumber * perGroupCount);

        service.setLock(new StampedLock());

        reduce("stampedLock", service::stampedLock);
        // stampedLock 方式, 共执行 20组，每组 100次，最长耗时 2541毫秒, 最短耗时 2419毫秒, 平均耗时 2497.800000毫秒
        // stampedLock 方式, 共执行 20组，每组 1000次，最长耗时 25272毫秒, 最短耗时 25085毫秒, 平均耗时 25215.000000毫秒
    }

    @ParameterizedTest
    @CsvSource({"20, 100", "20, 1000"})
    void reentrantLock(int groupCount, int perGroupCount) {
        groupNumber = groupCount;
        perGroupNumber = perGroupCount;
        timeCountList = Lists.newArrayListWithCapacity(groupNumber * perGroupCount);

        service.setReentrantLock(new ReentrantLock());
        reduce("reentrantLock", service::reentrantLock);

        // reentrantLock 方式, 共执行 20组，每组 100次，最长耗时 2550毫秒, 最短耗时 128毫秒, 平均耗时 1479.400000毫秒
        // reentrantLock 方式, 共执行 20组，每组 1000次，最长耗时 25592毫秒, 最短耗时 1362毫秒, 平均耗时 16253.050000毫秒
    }

    /**
     * 指定任务，并统计运行的时长
     *
     * @param taskName 任务名称
     * @param task     任务
     */
    private void reduce(String taskName, Runnable task) {

        // 对 runAndRecordTime 执行 groupNumber 次
        ThreadUtil.concurrencyTest(groupNumber, () -> runAndRecordTime(taskName, task));

        // @formatter:off
        System.err.printf("%s 方式, 共执行 %d组，每组 %d次，最长耗时 %d毫秒, 最短耗时 %d毫秒, 平均耗时 %f毫秒 \n",
            taskName, groupNumber, perGroupNumber, CollUtil.max(timeCountList), CollUtil.min(timeCountList),
            timeCountList.stream().collect(Collectors.averagingLong(Long::longValue)));
        // @formatter:on
    }

    /**
     * 指定任务，并运行。记录每组执行的耗时时长
     *
     * @param taskName 任务名称
     * @param task     任务
     */
    private void runAndRecordTime(String taskName, Runnable task) {
        TimeInterval timer = DateUtil.timer();
        for (int i = 0; i < perGroupNumber; i++) {
            task.run();
        }

        long time = timer.intervalMs();
        timeCountList.add(time);
        System.out.printf("%s 方式执行 %d次, 用时 %d 毫秒 \n", taskName, perGroupNumber, time);
    }

}
```
