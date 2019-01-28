---
title: 'Spring Cloud系列: Spring Boot Admin'
date: 2018-09-07 16:20:35
categories: Spring Cloud
tags: [Spring Cloud,监控,官方文档阅读,翻译]
toc: true
comments: true
---

本文主要介绍了Spring Boot Admin的使用，参考Spring Boot Admin 2.0.2版本(以下简称SBA，来自官方)官方文档，主要实现了其中案例，也包括一些自己的想法

文档地址：[http://codecentric.github.io/spring-boot-admin/current/](http://codecentric.github.io/spring-boot-admin/current/)

以下文章内容的例子都可以在我的[GitHub](https://github.com/GreedyPirate/Spring-Cloud-Stack)找到

# Spring Boot Admin介绍

## What

SBA是一个用于管理和监控Spring Boot项目的工具，包括线程，内存，Spring bean加载情况，日志等一系列可视化界面

## Why

熟悉Spring Boot的读者都知道Spring Boot actuator这款组件，它使用HTTP端点或JMX来管理和监控应用程序，但是没有提供图形化界面，仅仅提供了JSON格式的数据，同时无法做到集中管理应用，对运维十分不友好，SBA基于actuator不但解决了这些痛点，并且通过扩展实现了很多强大的功能，如日志级别动态更改，查看实时日志，查看URL映射等等，对管理微服务十分有意义

## How

环境的搭建将配合注册中心Eureka，当然也可以不使用注册中心，参考[Spring Boot Admin Server](http://codecentric.github.io/spring-boot-admin/current/#set-up-admin-server)一节,或使用别的注册中心，如[Consul](http://cloud.spring.io/spring-cloud-consul/)，Zookeeper，这些官方都已经在github给出了[案例](https://github.com/codecentric/spring-boot-admin/tree/master/spring-boot-admin-samples)

# 环境搭建

服务端和客户端均加入了spring-security组件，同时都配置了关闭请求拦截和跨域防范，详见笔者的GitHub[Spring-Cloud-Stack](https://github.com/GreedyPirate/Spring-Cloud-Stack)项目，[admin-server](https://github.com/GreedyPirate/Spring-Cloud-Stack/tree/master/admin-server)和[admin-client](https://github.com/GreedyPirate/Spring-Cloud-Stack/tree/master/admin-client)模块
**注意：** 按常理IDEA在勾选依赖生成项目之后，会加入bom版本管理，可是笔者也遇到了没有自动生成的情况，请读者注意pom文件是否有以下内容
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-dependencies</artifactId>
            <version>${spring-boot-admin.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## SBA 服务端

### pom依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-netflix-eureka-client</artifactId>
</dependency>
```

### 注册Eureka并添加@EnableAdminServer注解

```java
@SpringBootApplication
@EnableAdminServer
@EnableEurekaClient
public class AdminServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(AdminServerApplication.class, args);
	}
}
```

### spring-security配置
`anyRequest.permitAll`表示允许所有请求通过校验
`csrf.disable`表示关闭跨域防范

```java
@Configuration
public class SecurityPermitAllConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().permitAll()  
            .and().csrf().disable();
    }
}
```

### yml配置

简要说明: 主要配置端口，eureka，必须暴露所有web actuator断点，生产环境考虑到安全性，应当酌情开放，最后配置了spring-security的用户名和密码

```yml
server:
  port: 8115
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8000/eureka/,http://localhost:8001/eureka/
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
spring:
  security:
    user:
      name: user
      password: admin
```

## SBA 客户端

## pom依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>de.codecentric</groupId>
	<artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>
```

## Spring-Security配置

[同服务端](#spring-security配置)

## yml配置

这里提一个小坑点, server的地址必须加**http://** 前缀，否则会在启动日志中看到WARN，注册失败

```
server:
  port: 8116
spring:
  boot:
    admin:
      client:
        url: http://localhost:8115
  security:
    user:
      name: user
      password: admin
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```

# 最初效果

在启动注册中心，以及服务端，客户端之后，打开[http://localhost:8115](http://localhost:8115)，输入配置的用户名和密码即可登录


## 报错

java.io.IOException: Broken pipe, SBA 的issue中有回复：

This is a quite normal. The browser does some long polling and keeps the tcp connection open. If the browser window is closed the tcp connection is aborted and on the next write the exception is thrown. there is nothing to do about this, except changing the loglevel.
这是很正常的。浏览器执行一些长轮询并保持TCP连接打开。如果浏览器窗口关闭，则中止TCP连接，并在下一次写入时抛出异常。除了更改日志级别之外，这没有什么可做的。 


# 扩展

## UI配置

### 如何显示项目的版本号

```
info:
  version: @project.version@
```
此处的project.version引用了maven中的变量

效果图如下

<img src="https://ws4.sinaimg.cn/large/0069RVTdly1fv136xsz3yj31kw0b63zz.jpg" width="65%" align="left"/>

## 查看实时滚动日志

```
logging:
  file: client.log
```
配置日志文件位置即可，根据官方文档说明，SBA可以自动检测出url链接，同时支持日志颜色配置，但是项目启动时报错，遂放弃之
```
logging.pattern.file=%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(%5p) %clr(${PID}){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n%wEx
````

日志效果如下图：
<img src="https://ws3.sinaimg.cn/large/0069RVTdly1fv13clxp0qj31kw0d9wts.jpg" width="65%" align="left"/>

可以看到提供了下载按钮，其实是打开了一个网页页签，可复制出来，中文日志会出现乱码

## tag

tag可以给每一个客户端标识，有两种途径加入tag:

### 1. 元数据
```
spring:
  boot:
    admin:
      client:
        url: http://localhost:8115
        instance:
          metadata:
            tags:
              content: mesh
```

### 2. info端点
```
info:
  tags:
    title: mosi
```

<img src="https://ws2.sinaimg.cn/large/0069RVTdly1fv13qitypfj30x80mkjt8.jpg" width="65%" align="left"/>

**值得注意的是，两种方式的k-v表现形式, 第一个是tags.content为key，第二个是tags为key**

## 静态配置客户端

这一小节的内容单独用了两个项目，分别是[admin-static-client](https://github.com/GreedyPirate/Spring-Cloud-Stack/tree/master/admin-static-client)，[admin-static-server](https://github.com/GreedyPirate/Spring-Cloud-Stack/tree/master/admin-static-server)

通过Spring Cloud提供的静态配置，SBA支持静态配置client，首先建立客户端项目，只需要web，actuator两个依赖即可，

server端将Eureka依赖改为如下：
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter</artifactId>
</dependency>
```
接下来配置客户端的地址等信息
```
spring:
  cloud:
    discovery:
      client:
        simple:
          instances:
            admin-static-client:
              - uri: http://localhost:8117
                metadata:
                  management.context-path: /actuator
```
admin-static-client将是在界面上显示的客户端地址


# 提醒



## 邮件提醒

当注册在SBA server上的应用出现DOWN/OFFLINE等情况时，需要通过告警的方式告知运维人员，而邮件告警是常用的方式之一，SBA支持邮件告警，使用了spring-boot-mail组件来完成这一功能，需要在server端做以下工作：

**注: ** 以下邮件有关内容,通常情况需要获取授权码，以qq邮箱为例，请参照[http://service.mail.qq.com/cgi-bin/help?subtype=1&&id=28&&no=1001256](http://service.mail.qq.com/cgi-bin/help?subtype=1&&id=28&&no=1001256)获取授权码

### pom中加入依赖

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

### 需要的配置如下

```
  boot:
    admin:
      ui:
        title: "Spring Boot Admin监控管理中心"
      notify:
        mail:
          from: 发送方
          to: 收件方，多个逗号分隔
          cc: 抄送，多个逗号分隔
          template: classpath:/META-INF/spring-boot-admin-server/mail/status-changed.html # 定制邮件模板，请参考官方实现
  mail:
    host: smtp.qq.com
    port: 25
    username: 发送方用户名
    password: 授权码
    protocol: smtp
    test-connection: false
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
```

### 最终收到的邮件如图

将客户端下线之后，收到的邮件如下
<img src="https://ws4.sinaimg.cn/large/0069RVTdly1fv4hbx46zfj30we0lk41e.jpg" width="65%" align="left"/>

余下的第三方应用接入以及安全防护不再介绍，直接进入自定义通知

# 通知

当应用(实例)宕机时，服务端应该主动通知运维人员，达到告警的作用，在SBA中提供了这样的扩展，可以继承`AbstractEventNotifier`或`AbstractStatusChangeNotifier`，由于二者属于继承关系，读者直接继承AbstractStatusChangeNotifier即可

**注: **通知的方式有很多种，如钉钉，邮件，短信，大家按需扩展即可，以邮件举例，注入`JavaMailSender`对象即可实现邮件报警

下面给出一个告警样例代码

```java
@Component
public class MyNotifier extends AbstractStatusChangeNotifier {

    private Logger LOGGER = LoggerFactory.getLogger(this.getClass());

    public MyNotifier(InstanceRepository repositpry) {
        super(repositpry);
    }


    @Override
    protected Mono<Void> doNotify(InstanceEvent event, Instance instance) {

        return Mono.fromRunnable(() -> {
            if (event instanceof InstanceStatusChangedEvent) {
                StatusInfo statusInfo = ((InstanceStatusChangedEvent) event).getStatusInfo();
                String status = statusInfo.getStatus();
                Map<String, Object> details = statusInfo.getDetails();
                String detailStr = details.toString();
                boolean isOffline = statusInfo.isOffline();
                LOGGER.info("status info are: status:{}, detail:{}, isOffline:{}", status, detailStr, isOffline);

                String mavenVersion = instance.getBuildVersion().getValue();
                String healthUrl = instance.getRegistration().getHealthUrl();
                LOGGER.info("instance build version is {}, health check url is {}", mavenVersion, healthUrl);

                // 获取事件信息，instance(客户端)信息，包括前面说过的元信息，可用来发钉钉消息，短信等等的通知
                LOGGER.info("Instance {} ({}) is {}", instance.getRegistration().getName(), event.getInstance(),
                        ((InstanceStatusChangedEvent) event).getStatusInfo().getStatus());
            } else {
                LOGGER.info("Instance {} ({}) {}", instance.getRegistration().getName(), event.getInstance(),
                        event.getType());
            }
        });
    }
}
```

# 细节

最后聊聊一些细节内容，读者有需求的可深入了解
1. 对于监控的url请求，可以[添加header](http://codecentric.github.io/spring-boot-admin/current/#customizing-headers)，并对request，response[拦截](http://codecentric.github.io/spring-boot-admin/current/#customizing-instance-filter)
2. 使用2.0的服务端监控1.5版本的spring boot客户端，需要做一些[兼容处理](http://codecentric.github.io/spring-boot-admin/current/#monitoring-spring-boot-1.5.x)
3. 扩展UI，由于2.0使用了Vue.js重构，可以很方便的[扩展](http://codecentric.github.io/spring-boot-admin/current/#customizing-custom-views)

完结撒花









