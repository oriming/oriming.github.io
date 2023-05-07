---
title: 锁(Java synchronized)优化增强
author: Oriental Ming
date: 2023-05-07 10:04:00 +0800
categories: [锁, synchronized优化]
tags: [Lock, Java]
render_with_liquid: false
---

# Welcome

`synchronized`关键字往往是java中在处理高并发时的首选，尤其是**JDK6**之后，对synchronized不断的优化，提供了`三种锁的实现`，包括 `偏向锁 ⋙ 轻量级锁 ⋙ 重量级锁`，还提供了**自动的升级和降级机制**，使`synchronized`的性能得到极大的提升。
秉承废话不多说的规则，咱们开始...

----

## 1.原理 — 锁对象探究

|锁| 说明 |
|--|--|
| 对象 |  this、常量池对象 |
| 方法|  直接在方法上加`synchronized` |
| mark|  基于业务`mark`的对象 |

锁对象、方法的性能较差，因为在高并发情况下，所有的线程抢夺一个公共对象，虽然保证了线程安全，**但导致同时刻只能有一个线程执行锁定的方法。**

![通用锁对象的策略](/assets/img/2023-05-07-lock-synchronized-enhance-java/2023-05-07-11-36-03.png)

所以采用`业务级mark`的方式作为锁对象，将得到N倍性能的提升。原理是，对数据的读、写操作仅对`唯一标识mark 进行互斥`操作处理。
**例如：用户表主键id，100、101、102三个用户，当三个用户并发访问时，依据id的互斥性，100、101、102可同时执行业务逻辑。**

![锁mark](/assets/img/2023-05-07-lock-synchronized-enhance-java/2023-05-07-11-43-46.png)

**但Thread-1 和Thread-2不可同时执行业务逻辑**

![锁mark-notExec](/assets/img/2023-05-07-lock-synchronized-enhance-java/2023-05-07-11-44-29.png)

## 2.铺垫 — 研究示例

*模拟`用户表主键userId`的类型为 Integer，对其进行锁定。*

### 2.1锁Integer对象[-128,127]

```java
import cn.hutool.core.lang.Console;
import cn.hutool.core.thread.ThreadUtil;
import cn.hutool.core.util.RandomUtil;

import java.util.concurrent.TimeUnit;
import java.util.function.Consumer;

public class LockDemo {

    public static void main(String[] args) {
        LockDemo lockDemo = new LockDemo();
        // 测试
        lockDemo.concurrent(lockDemo::lockVar);
    }

    /**
     * 利用hutool提供的高并发测试类测试
     *
     * @param consumer 执行方法的业务逻辑
     */
    private void concurrent(Consumer<Integer> consumer) {
            // 测试的逻辑内容, 模拟50个线程并发执
        ThreadUtil.concurrencyTest(50, () -> consumer.accept(RandomUtil.randomInt(0, 5)));
    }

    /**
     * 锁对象变量, 避免拆装箱
     * <pre>
     *     失败，因为index对象的值在 [-128,127]之外, 锁的都是新对象，无效锁
     * </pre>
     *
     * @param index 变量
     */
    private void lockVar(int index) {
        synchronized (Integer.valueOf(index)) {
            content(index);
        }
    }

    /**
     * 模拟实际的业务逻辑
     *
     * @param index 标识, 例如: userId
     */
    private void content(Integer index) {
        Console.log("线程 {} 下标 {}, 进来了", Thread.currentThread().getName(), index);
        try {
            // 业务逻辑执行时间模拟
            TimeUnit.MILLISECONDS.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Console.error("线程 {} 下标 {}, 执行完毕", Thread.currentThread().getName(), index);
    }
}
```

**日志结果：**

![integer-log-cacheMark](/assets/img/2023-05-07-lock-synchronized-enhance-java/2023-05-07-11-47-52.png)

**结论：** 从结果看，已经实现了我们的目标要求，但是这仅仅局限于Integer范围在[-128,127]之间的整数，因为`Integer.valueOf(index)`方法使用了JVM的缓存机制。

```java
    public static Integer valueOf(int i) {
       // 当在[-128,127]范围内的时候，将使用缓存对象。当使用缓存对象的时候，多个线程并发抢夺一个对象，可实现锁的有效性。但超过这个范围那么锁将无效，请看下面示例
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

----

### 2.2锁Integer对象 非[-128,127]范围

执行的方法一样，唯独生成的随机数范围不一致

```java
    /**
     * 利用hutool提供的高并发测试类测试
     *
     * @param consumer 执行方法的业务逻辑
     */
    private void concurrent(Consumer<Integer> consumer) {
            // 超出JVM缓存，设置并发线程数为5个
        ThreadUtil.concurrencyTest(5, () -> consumer.accept(RandomUtil.randomInt(200, 205)));
    }
```

**日志结果：**

![integer-lock-failMark](/assets/img/2023-05-07-lock-synchronized-enhance-java/2023-05-07-11-50-24.png)

**结论：** 从日志分析发现，`相同下标对象同时执行了业务逻辑`，这说明锁已经失效。因为`超出JVM Integer的缓存`之后，都将`new`一个新对象，所以即使*下标对象的值一样，但是对象不一样*，导致各线程抢夺的对象不一致，故锁失效。

## 3.成果 — 隆重登场

> 现在我们有一个大前提，和一个条件。分别是：
> 前提： 将业务逻辑的【唯一标识(暂定mark)】作为锁对象
> 条件：【被锁的对象必须具有互斥性】。也就是说不同线程争夺的同一个`mark`必须是同一个对象。

**所以想到了利用缓存。既然JVM内部封装的Integer缓存机制可以实现，那么我们也可以自定义对象缓存机制实现！**

### 3.1源代码

```java
package priv.explore8.utils;

import com.google.common.collect.Sets;
import lombok.AccessLevel;
import lombok.NoArgsConstructor;

import java.util.Set;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Predicate;

/**
 * 高并发锁工具类
 *
 * @author Oriental Ming
 * @date 2023/5/7
 */
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class LockUtil {

    /**
     * mark缓存容器
     */
    private static final Set<Object> SET_CACHE = Sets.newConcurrentHashSet();

    /**
     * spinSet方法的锁对象
     */
    private static final Object SPIN_LOCK_OBJ = new Object();

    /**
     * 根据自旋的原理实现对特定对象的细粒度的锁处理，从而实现锁性能的提升
     * <pre>
     *    提升的性能是mark去重数量为N, 则提升N倍性能
     * </pre>
     *
     * @param mark 锁对象，譬如：userId, id, openid等唯一性标识性对象
     * @param run  自定义运行方法，根据业务需要可以更换成其他函数式接口。譬如：{@link Function}、{@link Predicate}...
     * @param <T>  锁对象泛型
     */
    public static <T> void spinSet(T mark, Consumer<T> run) {
        try {
            // @formatter:off
            for (;;) {
                // 锁住读、写，保证高并发的线程安全性
                synchronized (SPIN_LOCK_OBJ) {
                    // 当没有线程持有标记时，放行
                    if (!SET_CACHE.contains(mark)) {
                        SET_CACHE.add(mark);
                        break;
                    }
                }
            }

            run.accept(mark);
        } finally {
            // 释放
            SET_CACHE.remove(mark);
        }
    }
}

```

### 3.2测试

```java
package priv.explore8.utils;

import cn.hutool.core.lang.Console;
import cn.hutool.core.thread.ThreadUtil;
import cn.hutool.core.util.RandomUtil;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

class LockUtilTest {

    @Test
    void spinSet() {
        ThreadUtil.concurrencyTest(50, () -> {
            LockUtil.spinSet(RandomUtil.randomInt(200, 205), this::content);
        });
    }

    /**
     * 模拟实际的业务逻辑
     *
     * @param mark 标识, 例如: userId
     */
    private void content(Integer mark) {
        Console.log("线程 {} \t mark {}, \t 开始执行", Thread.currentThread().getName(), mark);
        try {
            TimeUnit.MILLISECONDS.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Console.error("线程 {} \t mark {}, \t 执行完毕", Thread.currentThread().getName(), mark);
    }
}
```

![testResult](/assets/img/2023-05-07-lock-synchronized-enhance-java/2023-05-07-12-07-29.png)

结论： **日志分析得出，此方式实现了我们之前提出的一个大前提和一个条件，从理论、实践上真真正正的实现了锁性能的提升！**
