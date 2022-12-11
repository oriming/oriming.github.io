---
title: SpringBoot Integration RabbitMQ
author: Sunny Boy
date: 2022-12-08 14:10:00 +0800
categories: [SpringBoot, RabbitMQ]
tags: [Spring]
render_with_liquid: false
---

SpringBoot 通过 AMQP 实现与 RabbitMQ 的集成。
基础的服务搭建工作就不介绍了，简要说一下 RabbitMQ 本地服务的构建，方便调试即可。

## 启动 RabbitMQ 服务

> 借助 Docker 容器的方便性，启动一个 RabbitMQ 容器

```shell
# 启动一个 RabbitMQ 服务。
# RabbitMQ Docker 官方描述: https://hub.docker.com/_/rabbitmq
# 设置用户名/密码（guest/guest），后续当与 SpringBoot 集成时需要
docker run -d -p 5672:5672 -p 15672:15672 --name some-rabbit -e RABBITMQ_DEFAULT_USER=guest -e RABBITMQ_DEFAULT_PASS=guest rabbitmq:management-alpine
```

👀 **特别说明：**

+ `5672:5672` RabbitMQ 容器监听的默认端口号。后续 SpringBoot 链接 RabbitMQ 端口就是这个！
+ `15672:15672` 是 RabbitMQ Web 管理页面暴露出的端口号，我们看一下官方解释

![官方解释](/assets/img/2022-12-10-SpringBoot/2022-12-11-20-44-40.png)

启动成功后，登陆管理平台验证即可，如下所示

![本地管理平台](/assets/img/2022-12-10-SpringBoot/2022-12-11-20-43-43.png)

## 正文开始

> 创建一个 SpringBoot 基础项目。
> 从 [Spring 官方构建](https://start.spring.io/) ，或者 IDEA 构建都可以。这里就不做赘述了 😜。

### 连接 RabbitMQ Broker

```java
package com.mq.config;

import com.rabbitmq.client.ShutdownSignalException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.Connection;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionListener;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.lang.NonNull;

import java.util.Collections;

/**
 * RabbitMQ 配置
 *
 * @author Sunny Boy
 * @date 2022/9/13 18:42
 */
@Slf4j
@Configuration
public class RabbitFactoryConfig {

 @Bean
 public ConnectionFactory connectionFactory() {
  // 创建连接工厂,获取MQ的连接。因为是本地服务，所以用 localhost
  CachingConnectionFactory connectionFactory = new CachingConnectionFactory("localhost", "5672");
  // 本地 RabbitMQ 服务的用户名和密码，后续可以集成到 application 配置文件中
  connectionFactory.setUsername("guest");
  connectionFactory.setPassword("guest");
  connectionFactory.setVirtualHost("/");

  connectionFactory.setConnectionListeners(Collections.singletonList(getConnectionListener()));
  return connectionFactory;
 }

 private ConnectionListener getConnectionListener() {
  return new ConnectionListener() {
   @Override
   public void onCreate(@NonNull Connection connection) {
    log.info("RabbitMQ 创建连接");
   }

   @Override
   public void onClose(@NonNull Connection connection) {
    log.info("RabbitMQ 关闭连接");
   }

   @Override
   public void onShutDown(@NonNull ShutdownSignalException signal) {
    log.error("RabbitMQ ShutDown! 详情: ", signal);
   }

   @Override
   public void onFailed(@NonNull Exception exception) {
    log.error("RabbitMQ Failed! 详情: ", exception);
   }
  };
 }
}
```

### Exchange 与 Queue 配置示例

```java
package com.common.mq.config;

import com.common.mq.constant.ExchangeConstant;
import com.common.mq.constant.QueueConstant;
import com.common.mq.constant.enums.RoutingKeyEnum;
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * RabbitMQ 交换机和队列配置
 *
 * @author Sunny Boy
 * @date 2022/9/13 19:04
 */
@Configuration
public class RabbitExchangeAndQueueConfig {

  // 交换机
  private static final String PUBLIC_EXCHANGE = "exchange.direct.public";
 private static final String DELAY_EXCHANGE = "exchange.direct.delay";
 private static final String DLX_EXCHANGE = "exchange.direct.dlx.application";

  // 路由
  public static final String APP1_KEY = "app1.routing.key";
 public static final String APP2_KEY = "app2.routing.key";
 public static final String DLX_PUBLIC_KEY = "dlx.public.routing.key";

  // 队列
  public static final String APP1_QUEUE = "queue.direct.app1";
 public static final String APP2_QUEUE = "queue.direct.app2";

 @Bean
 public DirectExchange publicExchange() {
  return ExchangeBuilder.directExchange(PUBLIC_EXCHANGE).build();
 }

 /**
  * app1 消息发送队列，并指定死信队列
  *
  * @return 队列
  */
 @Bean
 public Queue app1Queue() {
  Map<String, Object> params = new ConcurrentHashMap<>(4);
  // 指定死信交换器
  params.put("x-dead-letter-exchange", DLX_EXCHANGE);
  // 指定死信队列
  params.put("x-dead-letter-routing-key", DLX_PUBLIC_KEY.getCode());
  return QueueBuilder.durable(APP1_QUEUE).withArguments(params).build();
 }

 /**
  * app2 消息发送队列，并指定死信队列
  *
  * @return 队列
  */
 @Bean
 public Queue app2Queue() {
  Map<String, Object> params = new ConcurrentHashMap<>(4);
  // 指定死信交换器
  params.put("x-dead-letter-exchange", DLX_EXCHANGE);
  // 指定死信队列
  params.put("x-dead-letter-routing-key", DLX_PUBLIC_KEY.getCode());
  return QueueBuilder.durable(APP2_QUEUE).withArguments(params).build();
 }

 @Bean
 public Binding bindingApp1Queue() {
  return BindingBuilder.bind(app1Queue()).to(publicExchange()).with(APP1_KEY.getCode());
 }

 @Bean
 public Binding bindingApp2Queue() {
  return BindingBuilder.bind(app2Queue()).to(publicExchange()).with(APP2_KEY.getCode());
 }
}
```
