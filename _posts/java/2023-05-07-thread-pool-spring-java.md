---
title: Spring线程池配置
author: Oriental Ming
date: 2023-05-07 15:20:00 +0800
categories: [Spring, 线程池配置]
tags: [Java, Spring]
render_with_liquid: false
---

# Welcome

提供一个可供参考的线程池配置。重点是对**线程数量的配置**和**拒绝策略**的制定。

## 线程数配置策略

**线程池的理想大小取决于被提交任务的类型以及所部属系统的特性。** 在代码中通常不会固定线程池的大小，而应该通过某种配置机制来
提供，或者根据 `Runtime.availableProcessors` 来动态计算。

幸运的是，要设置线程池的大小也并不困难，只需要避免“过大” 和 “过小” 这两种极端情况。

+ 如果线程池过大，那么大量的线程将在相对很少的CPU和内存资源上发生竞争，这不仅会导致更高的内存使用量，而且还可能耗尽资源。
+ 如果线程池过小，那么将导致许多空闲的处理器无法执行工作，从而降低吞吐率。

**要想正确的配置线程池的大小，必须分析计算环境、资源预算和任务的特性。** 在部署的系统中有多少个CPU？多大的内存？任务是
计算密集型、I/O密集型还是二者皆可？他们是在需要像JDBC连接这样的稀缺资源？如果需要执行不同类别的任务，并且他们之间的行为
相差很啊，那么应该考虑使用多个线程池，从而使每个线程池可以根据各自的工作负载来调整。

**要正确的设置线程池的大小，你必须估算出任务的等待时间与计算时间的比值。** 这种估算不需要很精确，并且可以通过一些分析或
监控工具来获得。
你还可以通过另一种方法来调节线程池的大小：**在某个基准负载下，分别设置不同大小的线程池来运行应用程序，并观察CPU利用率的水平。**

给定下列定义：

+ N<sub>cpu</sub> = number of CPUs (CPU 数量)
+ U<sub>cpu</sub> = target CPU utilization (CPU 目标利用率)，0 <= U<sub>cpu</sub> <=1
+ W/C = ratio of wait time to compute time (等待时间与计算时间的比率)

要使处理器达到期望的使用率，线程池的最优大小等于：

&emsp; &emsp; &emsp;  ***N<sub>threads</sub> = N<sub>cpu</sub> \* U<sub>cpu</sub> \* (1+ W/C)***

可以通过 `Runtime` 来获得 CPU 的数量：

```java
int N_CPUS = Runtime.getRuntime().availableProcessors();
```

注意，CPU 周期并不是唯一影响线程池大小的资源，还包括内存、文件句柄、套接字句柄和数据库连接等。计算这些资源对线程池的
约束条件：计算每个任务对该资源的需求量，然后用该资源的可用总量除以每个任务的需求量，所得结果就是线程池大小的上限！

+ 计算密集型的任务可以设定：**N<sub>threads</sub> = N<sub>cpu</sub> + 1**，通常能实现最优的利用率。即使当计算密集型的线程
  偶尔由于页缺失故障或者其他原因而暂停时，这个 “额外” 的线程也能确保 CPU 的时钟周期不会被浪费。

+ I/O操作任务或其他阻塞操作的任务，由于线程并不会一直执行，因此线程池的规模应该更大。

------------ **以上内容摘自 《Java 并发编程实战》 第8章第2节**  ------------

## 示例配置

```java
package com.github.config;

import cn.hutool.core.thread.ThreadFactoryBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.ThreadFactory;

/**
 * 线程池配置
 *
 * @author Oriental Ming
 */
@Configuration
public class ThreadPoolTaskConfig {

    /**
     * 定时任务之核心线程数量
     */
    private static final int CORE_POOL_SIZE;
    /**
     * 最大线程池的数量默认是机器逻辑核数量
     */
    private static final int MAXIMUM_POOL_SIZE;
    /**
     * 线程存活时间，单位秒
     */
    private static final int KEEP_ALIVE_TIME = 60;
    /**
     * 队列的长度
     */
    private static final int QUEUE_CAPACITY = 100;
    /**
     * 线程工厂名称
     */
    private static final ThreadFactory FACTORY;

    /**
     * 线程池命名前缀
     */
    private static final String THREAD_NAME_PREFIX = "self-worker-";

    static {
        int computerCoreSize = Runtime.getRuntime().availableProcessors();
        CORE_POOL_SIZE = computerCoreSize << 1;
        MAXIMUM_POOL_SIZE = computerCoreSize << 2;
        FACTORY = new ThreadFactoryBuilder().setNamePrefix(THREAD_NAME_PREFIX).setDaemon(true).build();
    }

    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor poolTaskExecutor = new ThreadPoolTaskExecutor();
        poolTaskExecutor.setCorePoolSize(CORE_POOL_SIZE);
        poolTaskExecutor.setMaxPoolSize(MAXIMUM_POOL_SIZE);
        poolTaskExecutor.setKeepAliveSeconds(KEEP_ALIVE_TIME);
        poolTaskExecutor.setQueueCapacity(QUEUE_CAPACITY);
        poolTaskExecutor.setThreadNamePrefix(THREAD_NAME_PREFIX);
        poolTaskExecutor.setThreadFactory(FACTORY);
        // 当缓存队列和MaxPoolSize达到上限后，执行自定义拒绝策略
        poolTaskExecutor.setRejectedExecutionHandler(new MyRejectedExecutionHandler());
        poolTaskExecutor.setAllowCoreThreadTimeOut(true);

        poolTaskExecutor.initialize();
        return poolTaskExecutor;
    }

}
```

## 自定义拒绝策略

```java
package com.github.config;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.atomic.LongAdder;

/**
 * 自定义线程池拒绝策略
 * <pre>
 *     创建新线程去执行任务
 * </pre>
 *
 * @author Oriental Ming
 */
@Slf4j
public class MyRejectedExecutionHandler implements RejectedExecutionHandler {
    /**
     * 记录新增线程数量
     */
    private static final LongAdder NUMBER = new LongAdder();
    /**
     * 记录新增线程数量
     */
    private static final LongAdder RESET_NUMBER = new LongAdder();
    /**
     * 拒绝策略新增线程数量记录单位20, 每新增20重新记录
     * <pre>
     *     总新增线程数计算公式: RESET_NUMBER * 20 + (NUMBER+1)
     * </pre>
     */
    private static final int MAX_LIMIT = 19;

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        int currentNumber = NUMBER.intValue();
        if (currentNumber >= MAX_LIMIT) {
            NUMBER.reset();
            RESET_NUMBER.increment();
            log.info("线程池拒绝策略, 新增线程计数重置第{}次", RESET_NUMBER.intValue());
        }

        NUMBER.increment();
        new Thread(r, "new-add-Thread-" + currentNumber).start();
        log.info("触发线程池拒绝策略, 新增线程, 编号: {}", currentNumber);
    }
}

```
