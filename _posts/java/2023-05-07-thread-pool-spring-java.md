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

`在配置线程池参数时，理清楚在使用线程池中线程的场景，是属于CPU密集计算型，还是磁盘I/O。`

`SERVER_LOGIC_CORE` :  服务器的逻辑核心数；<br>
`CORE_POOL_SIZE`：线程池配置的核心线程数；

+ 如果CPU密集计算型，CORE_POOL_SIZE = SERVER_LOGIC_CORE 。理由：此类型线程忙碌。
+ 如果是磁盘I/O型，CORE_POOL_SIZE = 2 * SERVER_LOGIC_CORE 。理由：此类型线程大多时候是处于等待，空置的时候较多。

以上仅是参考性配置，具体详细的配置还需要依赖不同的业务场景和代码逻辑。

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
