---
title: Java 并行 Util
author: Oriental Ming
date: 2023-05-11 21:14:00 +0800
categories: [Java, 并行 Util]
tags: [Java, Parallel]
render_with_liquid: false
---

# Welcome

在实际的生产开发环境中，有时会遇到需要**简易**并行处理的场景。譬如, 多项任务之间有几点共性：

+ 具有相同/类型的返回结果
+ 任务与任务具备隔离性（相互没有影响）
+ 每项任务的执行都比较耗时

示例如下：

```java
// 批量保存 xxxx 对象。 耗时：5s
boolean res1 = service1.saveBatch(xxxx);
// 批量更新 yyyy 对象。耗时：4s
boolean res2 = service2.updateBatchByIds(yyyy);
// 通过平台(HTTP)发送消息，并等待结果。耗时：6s
boolean res3 = service3.sendMsg(jjjj);
// 通过 RPC 向其他 Module 操作，并等待结果。耗时：3s
boolean res4 = service4.rpcOperate(jjjj);
```

此时，我们既可以针对此情景，异步执行每一项任务，并在最后收集结果。

**最终的目的：**扩大资源利用率，减少任务列表执行的总耗时。 如上示例串行需要耗时 18秒，而并行只需要 6秒左右！

## 1. 依赖

```xml
   <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.24</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>31.1-jre</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.9.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

## 2. 源码

```java
package org.example.starter.util;

import com.google.common.util.concurrent.ListeningExecutorService;
import com.google.common.util.concurrent.MoreExecutors;
import lombok.AccessLevel;
import lombok.NoArgsConstructor;
import lombok.SneakyThrows;

import java.util.*;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.function.Supplier;

/**
 * 并行工具类。
 * <pre>
 *     适用：并行的任务中 I/O 耗时长的场景
 *       1. 具有相同/类型的返回结果
 *       2. 任务与任务具备隔离性（相互没有影响）
 *       3. 每项任务的执行都比较耗时
 * </pre>
 *
 * @author Oriental Ming
 * @date 2023/5/11 19:58
 */
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class ParallelUtil {

    /**
     * 初始化公共的缓存线程池。
     * 如果任务特别多，耗时过长，队列时间太长容易出现长时间不执行的问题
     */
    private static final ExecutorService CACHED_THREAD_POOL = Executors.newCachedThreadPool();

    /**
     * 对任务列表中的每一项任务异步执行，并收集每一项任务的执行结果（自研）
     *
     * @param taskList 任务列表
     * @param <T>      对象
     * @return 任务结果集
     */
    @SneakyThrows
    public static <T> Set<T> execAsyncBatchByMine(Collection<Supplier<T>> taskList) {
        // 异步执行，并收集异步过程
        LinkedList<Future<T>> collectFuture = new LinkedList<>();
        for (Supplier<T> supplier : taskList) {
            // 借用 Hutool 封装
            Future<T> future = CACHED_THREAD_POOL.submit(supplier::get);
            collectFuture.add(future);
        }

        // 循环获取结果
        Set<T> result = new HashSet<>();

        // 阻塞循环收集并行集合的结果
        int index = 0;
        while (!collectFuture.isEmpty()) {
            Future<T> future = collectFuture.get(index);

            if (future.isDone()) {
                // 归并异步执行的结果
                result.add(future.get());
                // 删除已执行完毕的任务（底层原理是链表的解链）
                collectFuture.remove(index);

                // 如果有执行完毕的，就从头开始检查
                index = 0;
                continue;
            }

            // 如果循环完成后，任务存在还没有执行完的，则从头开始
            if (collectFuture.size() == ++index) {
                index = 0;
            }
        }

        return result;
    }


    /**
     * 对任务列表中的每一项任务异步执行，并收集每一项任务的执行结果（Guava 实现）
     *
     * @param taskList 任务列表
     * @param <T>      对象
     * @return 任务结果集
     */
    @SneakyThrows
    public static <T> Set<T> execAsyncBatchByGuava(Collection<Callable<T>> taskList) {
        ListeningExecutorService executorService = MoreExecutors.listeningDecorator(CACHED_THREAD_POOL);
        List<Future<T>> futures = executorService.invokeAll(taskList);

        Set<T> result = new HashSet<>();
        for (Future<T> future : futures) {
            result.add(future.get());
        }

        return result;
    }
}
```

## 3. Unit Test

### 3.1 Abstract class

```java
package org.example;

import cn.hutool.core.util.RandomUtil;
import lombok.SneakyThrows;

import java.util.concurrent.TimeUnit;
import java.util.function.Consumer;

/**
 * 公共的异步测试类
 *
 * @author Oriental Ming
 * @date 2023/5/11 20:18
 */
public abstract class AbstractAsyncTest {

    @SneakyThrows
    protected int sleepTime(int o, Consumer<String> consumer) {
        int sleepMillSec = RandomUtil.randomInt(3000, 7000);
        Thread thread = Thread.currentThread();

        String printMsg = String.format("对象: %d，\t线程ID: %d, \t线程name: %s,\t 延迟 %d 毫秒 \n", o, thread.getId(), thread.getName(), sleepMillSec);
        consumer.accept(printMsg);

        TimeUnit.MILLISECONDS.sleep(sleepMillSec);
        return o;
    }
}

```

### 3.2 ParallelUtil Test

```java
package org.example.starter.util;

import org.example.AbstractAsyncTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.Callable;
import java.util.function.Supplier;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

class ParallelUtilTest extends AbstractAsyncTest {


    List<Supplier<Integer>> params1;
    List<Callable<Integer>> params2;

    @BeforeEach
    void setUp() {
        params1 = new ArrayList<>(6);
        params2 = new ArrayList<>(6);

        for (int i = 0; i < 6; i++) {
            int finalI = i;
            params1.add(() -> sleepTime(finalI, System.out::println));
            params2.add(() -> sleepTime(finalI, System.out::println));
        }
    }

    /**
     * 串行执行（作为结果的对比）
     */
    @Test
    void serialExec() {

        long start = System.currentTimeMillis();
        Set<Integer> result = Sets.newHashSet();
        for (Supplier<Integer> supplier : params1) {
            result.add(supplier.get());
        }
        System.err.printf("----------总耗时 %d 毫秒------------ \n", (System.currentTimeMillis() - start));

        assertNotNull(result);
        System.out.println(result);
        assertEquals(6, result.size());
    }

    @Test
    void execAsyncBatchByMine() {
        long start = System.currentTimeMillis();
        Set<Integer> result = ParallelUtil.execAsyncBatchByMine(params1);
        System.err.printf("----------总耗时 %d 毫秒------------ \n", (System.currentTimeMillis() - start));

        assertNotNull(result);
        System.out.println(result);
        assertEquals(6, result.size());
    }

    @Test
    void execAsyncBatchByGuava() {
        long start = System.currentTimeMillis();
        Set<Integer> result = ParallelUtil.execAsyncBatchByGuava(params2);
        System.err.printf("----------总耗时 %d 毫秒------------ \n", (System.currentTimeMillis() - start));

        assertNotNull(result);
        System.out.println(result);
        assertEquals(6, result.size());
    }
}
```

## 4. 测试结果

### 4.1 serialExec Method

```java
对象: 0，    线程ID: 1,     线程name: main,     延迟 6874 毫秒

对象: 1，    线程ID: 1,     线程name: main,     延迟 3484 毫秒

对象: 2，    线程ID: 1,     线程name: main,     延迟 4127 毫秒

对象: 3，    线程ID: 1,     线程name: main,     延迟 3917 毫秒

对象: 4，    线程ID: 1,     线程name: main,     延迟 4705 毫秒

对象: 5，    线程ID: 1,     线程name: main,     延迟 5731 毫秒

----------总耗时 28874 毫秒------------
[0, 1, 2, 3, 4, 5]
```

### 4.2 execAsyncBatchByMine Method

```java

对象: 1，    线程ID: 17,     线程name: pool-1-thread-2,     延迟 6475 毫秒

对象: 0，    线程ID: 16,     线程name: pool-1-thread-1,     延迟 5831 毫秒

对象: 2，    线程ID: 18,     线程name: pool-1-thread-3,     延迟 4535 毫秒

对象: 3，    线程ID: 19,     线程name: pool-1-thread-4,     延迟 3002 毫秒

对象: 4，    线程ID: 20,     线程name: pool-1-thread-5,     延迟 5289 毫秒

对象: 5，    线程ID: 21,     线程name: pool-1-thread-6,     延迟 6412 毫秒

----------总耗时 6495 毫秒------------
[0, 1, 2, 3, 4, 5]

```

### 4.3 execAsyncBatchByGuava Method

```java
对象: 2，    线程ID: 16,     线程name: pool-1-thread-1,     延迟 5643 毫秒

对象: 4，    线程ID: 18,     线程name: pool-1-thread-3,     延迟 6600 毫秒

对象: 3，    线程ID: 20,     线程name: pool-1-thread-5,     延迟 5497 毫秒

对象: 5，    线程ID: 19,     线程name: pool-1-thread-4,     延迟 5708 毫秒

对象: 0，    线程ID: 17,     线程name: pool-1-thread-2,     延迟 4110 毫秒

对象: 1，    线程ID: 21,     线程name: pool-1-thread-6,     延迟 5167 毫秒

----------总耗时 6612 毫秒------------
[0, 1, 2, 3, 4, 5]

```
