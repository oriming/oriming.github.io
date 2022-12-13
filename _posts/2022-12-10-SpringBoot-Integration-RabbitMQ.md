---
title: SpringBoot Integration RabbitMQ
author: Sunny Boy
date: 2022-11-08 14:10:00 +0800
categories: [SpringBoot, RabbitMQ]
tags: [Spring, RabbitMQ]
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

1. 命令说明

+ `5672:5672` RabbitMQ 容器监听的默认端口号。后续 SpringBoot 链接 RabbitMQ 端口就是这个！
+ `15672:15672` 是 RabbitMQ Web 管理页面暴露出的端口号，我们看一下官方解释

![官方解释](/assets/img/2022-12-10-SpringBoot/2022-12-11-20-44-40.png)

启动成功后，登陆管理平台验证即可，如下所示

![本地管理平台](/assets/img/2022-12-10-SpringBoot/2022-12-11-20-43-43.png)

2. 插件安装

+ ① 下载
前往官方 [官方插件下载地址](<https://www.rabbitmq.com/community-plugins.html>) 下载 `rabbitmq_delayed_message_exchange` 插件到本地，文件后缀 `.ez`

  ⚠️ **注意：找到正确的插件版本。** 可以在 Docker 容器中执行 `rabbitmqctl --version` 获取 RabbitMQ 的版本信息，以此作为依据下载对应的插件版本!

+ ② 导入到指定目录

  进入容器，查看插件安装目录

  ```bash
  # 登陆 Docker 容器：
  docker exec -it some-rabbit bash

  # 查询插件安装目录
  rabbitmq-plugins directories
    Listing plugin directories used by node rabbit@ea5925747aaf
    Plugin archives directory: /opt/rabbitmq/plugins
    Plugin expansion directory: /var/lib/rabbitmq/mnesia/rabbit@ea5925747aaf-plugins-expand
    Enabled plugins file: /etc/rabbitmq/enabled_plugins

  # 可以看到，插件的安装目录是：/opt/rabbitmq/plugins，所以把插件上传到此目录中
  ```

  通过 `docker cp` 指令传输插件包到 `/opt/rabbitmq/plugins`

  ```bash
  # 打开电脑主机终端，传输插件到容器指定目录
  docker cp rabbitmq_delayed_message_exchange-3.11.1.ez some-rabbit:/opt/rabbitmq/plugins

  # 开启插件
  rabbitmq-plugins enable rabbitmq_delayed_message_exchange

  # 验证是否开启成功
  rabbitmq-plugins list | grep delayed

  # 只要第一列显示 [E*] 就证明安装并启动成功
  [E*] rabbitmq_delayed_message_exchange 3.11.1
  ```

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

{: .nolineno }

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

{: .nolineno }

### 消息发送 Service

```java
package priv.component.service;

import org.springframework.lang.NonNull;

/**
 * 消息发送
 *
 * @author Sunny Boy
 * @date 2022/9/14 09:09
 */
public interface MsgSender {

    /**
      * 向指定的路由发送消息
      *
      * @param routingKey 路由key
      * @param message    消息内容
      */
    void send(@NonNull String routingKey, @NonNull String message);

    /**
      * RabbitMQ 实现延时器任务。到时之后会自动转到 {@link priv.component.constant.QueueConstant#DLX_DELAY_QUEUE} 队列
      *
      * @param message   消息内容
      * @param delayTime 消息延时时间，单位：毫秒
      */
    void delay(@NonNull String message, @NonNull int delayTime);
}
```

### 消息消费

```java
package priv.component.consumer;

import cn.hutool.json.JSONUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;
import org.springframework.util.Assert;
import priv.component.constant.QueueConstant;
import priv.component.exception.MqMessageException;
import priv.component.model.dto.MqMessage;

/**
 * Sd APP 接收器
 *
 * @author Sunny Boy
 * @date 2022/11/12 10:25
 */
@Slf4j
@Component
public class SdAppConsumer {

    @RabbitListener(queues = QueueConstant.SD_APP_QUEUE)
    public void receiver(String message) {
        log.info("接收的消息: {}", message);
        boolean typeJSON = JSONUtil.isTypeJSON(message);
        if (!typeJSON) {
        throw new MqMessageException("RabbitMQ 消息异常");
        }

        MqMessage body = JSONUtil.toBean(message, MqMessage.class);
        Assert.notNull(body, "sd app consumer 接收的消息异常");
    }

}
```

## 测试

启动服务，发起集成测试，验证消息是否成功订阅：

```bash
2022-12-13 21:38:14.094  INFO 14686 --- [ntContainer#2-1] priv.component.consumer.SdAppConsumer    : 接收的消息: {"resultEnum":"SUCCESS","typeEnum":"PORT_VALUE","content":"test"}
```

**[演示代码 Github 地址！！！](https://github.com/3bluebird/SpringBoot-Integration-RabbitMQ)**
