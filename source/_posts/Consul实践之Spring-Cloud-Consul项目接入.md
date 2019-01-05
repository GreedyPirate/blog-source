---
title: Consul实践之Spring Cloud Consul项目接入
date: 2018-11-19 15:03:53
categories: 微服务注册中心
tags: [consul,注册中心]
toc: true
comments: true
---

>本文主要介绍Spring Cloud对consul的支持，分为消费者和生产者两个客户端

## 应用详情

分别新建两个spring-boot项目

| 应用名称         | 端口 | consul注册地址    |
| ---------------- | ---- | ----------------- |
| consumer-service | 8301 | 10.9.181.34:8500  |
| producer-service | 8302 | 10.9.117.128:8500 |

## consul接入

### 所需依赖

除了通用的web模块，主要需要consul-discovery与actuator，前者用于consul客户端接入，后者提供健康检查接口

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 启动类
```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsulConsumerApplication {
	public static void main(String[] args) {
		SpringApplication.run(ConsulConsumerApplication.class, args);
	}
}
```

### 配置

```
server:
  port: 8301
spring:
  application:
    name: consumer-service
  cloud:
    consul:
      host: localhost #consul客户端地址
      port: 8500
      retry:
        max-attempts: 3
        initial-interval: 1000
        max-interval: 2000
        multiplier: 1.1
      discovery:
        health-check-interval: 10s #健康检查默认时间间隔
        health-check-path: /actuator/health #健康检查默认请求路径
        health-check-timeout: 5s #超时时间
        #为服务生成一个32位随机字符作为实例名，并非最佳实践
        #instance-id: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
        health-check-tls-skip-verify: true #跳过https校验
        service-name: consumer-service
        heartbeat: 
          enabled: true
          ttl-value: 5
          ttl-unit: s
        prefer-ip-address: true #显示真实ip，而不是主机名
```
















