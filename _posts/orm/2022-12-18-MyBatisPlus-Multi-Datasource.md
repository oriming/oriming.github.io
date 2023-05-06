---
title: MyBatis-Plus 多数据源
author: Oriental ming
date: 2022-12-18 18:09:00 +0800
categories: [ORM, MyBatis-Plus]
tags: [ORM, MyBatis-Plus]
render_with_liquid: false
---

> 多数据源的配置方式有多种，既可以通过注解（@DS），也可以通过拦截器的方式处理。不同的方式解决的业务问题领域不一致而已。
> 本文介绍的方式是使用 **混合配置** 的方式实现，原理是：**通过拦截器，依据类所属的包名动态切换数据源！**

## 1. 引入必备依赖

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
  <!-- 选择与项目中 baomidou 框架对应的版本-->
  <version>${version}</version>
</dependency>
```

## 2. Yaml 文件配置

> 提醒：可以把此 `yaml` 文件抽离为公共配置文件，在其他的配置文件中显性引用。

```yaml

# 自定义配置。方便
test:
  datasource:
    alias:
      applet: &AliasApplet applet
      system: &AliasSystem system
    # 数据库的用户名和密码
    account:
      username: test
      password: root
    url:
      applet: jdbc:mysql://127.0.0.1:3306/test_applet?useSSL=false&useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
      system: jdbc:mysql://127.0.0.1:3306/test_system?useSSL=false&useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    # 配置顺序依据优先原则, 小范围配置在前。如果不配置则使用默认数据源
    relationship:
      mapping:
        # key：指定的datasource别名，value：需要切换的类路径 包名
        *AliasApplet : "com.test.app"
        *AliasSystem : "com.test.system"

spring:
  datasource:
    dynamic:
      # 默认数据源
      primary: ${test.datasource.alias.applet}
      #  启用严格匹配数据源,未匹配到指定数据源时抛异常
      strict: true
      datasource:
        # 譬如：小程序专用库
        *AliasApplet :
          driver-class-name: com.mysql.cj.jdbc.Driver
          url: ${test.datasource.url.applet}
          username: ${test.datasource.account.username}
          password: ${test.datasource.account.password}
        # 譬如：后台管理专用库
        *AliasSystem :
          driver-class-name: com.mysql.cj.jdbc.Driver
          url: ${test.datasource.url.system}
          username: ${test.datasource.account.username}
          password: ${test.datasource.account.password}
```

## 3. Spring 拦截器配置

+ 读取 `yaml`

```java
package com.test.component;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.Map;

/**
 * 数据源与包关系配置类, 配置有顺序要求, 依据小范围优先配置原则
 *
 * @author Oriental ming
 * @date 2021/7/15
 */
@Data
@Component
@ConfigurationProperties(prefix = "test.datasource.relationship")
public class DataSourceProperty {

    /**
     * <pre>
     *     包名和数据源的对应关系
     *     key: 数据源名, value: 包名
     * </pre>
     */
    private Map<String, String> mapping;

}
```

----

+ 拦截

```java
package com.test.config.impl;

import com.baomidou.dynamic.datasource.toolkit.DynamicDataSourceContextHolder;
import com.test.component.DataSourceProperty;
import org.springframework.context.annotation.Configuration;
import org.springframework.lang.NonNull;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashMap;
import java.util.Map;

/**
 * 数据源切换拦截器
 *
 * @author Oriental ming
 * @date 2021/7/15
 */
@Configuration
public class DataSourceInterceptor implements HandlerInterceptor {

    private final Map<String, String> PACK_MAP_DATASOURCE;
    private final Map<Object, String> BEAN_MAP_PACK = new HashMap<>();

    public DataSourceInterceptor(DataSourceProperty dataSourceProperty) {
        // 如果没有配置数据源关系, 则使用默认数据源
        PACK_MAP_DATASOURCE = dataSourceProperty.getMapping();
    }

    @Override
    public boolean preHandle(@NonNull HttpServletRequest request, @NonNull HttpServletResponse response, @NonNull Object handler) {
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }

        String packageName = getPackageName((HandlerMethod) handler);
        // 依据包路径动态配置数据源, 如果没有匹配的则使用默认数据源
        for (Map.Entry<String, String> entry : PACK_MAP_DATASOURCE.entrySet()) {
            if (packageName.contains(entry.getValue())) {
                DynamicDataSourceContextHolder.push(entry.getKey());
                break;
            }
        }

        return true;
    }

    @Override
    public void afterCompletion(@NonNull HttpServletRequest request, @NonNull HttpServletResponse response, @NonNull Object handler, Exception ex) {
        // 请求完成，及时清理数据源配置
        DynamicDataSourceContextHolder.clear();
    }

    /**
      * 依据方法名称获取其包的全名
      *
      * @param method 受理的方法
      * @return 方法所在类的包全名
      */
    private String getPackageName(HandlerMethod method) {
        Object bean = method.getBean();
        if (BEAN_MAP_PACK.containsKey(bean)) {
            return BEAN_MAP_PACK.get(bean);
        }

        String packageName = method.getMethod().getDeclaringClass().getName();
        BEAN_MAP_PACK.put(bean, packageName);
        return packageName;
    }
}
```

----

+ 配置 WebMvcConfigurer

```java
package com.test.config;

import com.test.config.impl.DataSourceInterceptor;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * 公众号端MVC全局配置
 *
 * @author Oriental ming
 * @date 2021/7/15
 */
@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final DataSourceInterceptor dataSourceInterceptor;

    /**
     * 数据源切换
     *
     * @param registry 拦截器注册类
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(dataSourceInterceptor).addPathPatterns("/**");
    }
}
```
