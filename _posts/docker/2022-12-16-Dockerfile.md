---
title: Dockerfile
author: Orient ming
date: 2022-12-16 16:55:00 +0800
categories: [Docker, IDEA]
tags: [Docker, jar]
render_with_liquid: false
---

通过 SpringBoot 官网建议，以及项目经验，总结出的一版 Dockerfile 文档模版。

## 构建

```Dockerfile
# 我们一般使用 alpine 的JDK，以此减小镜像的体积
FROM openjdk:8-jdk-alpine
LABEL maintainer="Orient ming"

# SpringBoot 官方推荐，最小权限原则，考虑到安全
ARG USERNAME=spring
ARG JAR_FILE=target/*.jar

RUN addgroup -S ${USERNAME} \
    && adduser -S ${USERNAME} -G ${USERNAME} \
    && ln -sf /usr /share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

USER ${USERNAME}:${USERNAME}
# 也是 jar 运行的日志文件保存目录
WORKDIR /home/${USERNAME}

# 指定内存，并增强随机数获取逻辑
ENV JAVA_OPTS="-Xms100m -Xmx100m -Djava.security.egd=file:/dev/./urandom"

COPY ${JAR_FILE} app.jar

EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -jar /app.jar ${0} ${@}"]

```

`docker run` 的运行示例:

```shell
# 可以指定服务的端口号
docker run -p 9000:9000 myorg/myapp --server.port=9000
# 可以自定义 JVM 的配置信息
docker run -p 8080:8080 -e "JAVA_OPTS=-Ddebug -Xmx128m" myorg/myapp
```

## IDEA 配置

### 链接远程服务器

![远程服务器连接](/assets/img/2022-12-16-Dockerfile/2022-12-16-16-58-15.png)

### 配置 Dockerfile

![配置](/assets/img/2022-12-16-Dockerfile/2022-12-16-16-58-57.png)
