---
title: Spring boot实践之事件监听
date: 2018-07-31 15:53:26
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---
在Spring Boot doc的*Application Events and Listeners*一章中提到，Spring Boot按以下顺序提供了6个事件，供开发者编写`ApplicationListener`监听相应的事件

	1.ApplicationStartingEvent：在开始运行时，监听器注册和初始化之后被触发
	2.ApplicationEnvironmentPreparedEvent：发现 Environment 被上下文使用时，上下文被创建之前触发
	3.ApplicationPreparedEvent：在启动刷新之前，bean定义被加载之后被触发
	4.ApplicationStartedEvent：上下文刷新之前，应用和命令行启动器运行之前触发
	5.ApplicationReadyEvent：在所有应用和命令行启动器调用之后，这表明应用已经准备好处理请求
	6.ApplicationFailedEvent：启动时出现异常触发
	
## 代码

编写代码的难度不高，读者可根据自己的需求编写相应的listener，以ApplicationStartingEvent为例

```
@Component
public class SpringBootListener implements ApplicationListener<ApplicationStartingEvent> {
    @Override
    public void onApplicationEvent(ApplicationStartingEvent event) {
        System.out.println("listening spring boot starting event");
    }
}

```

## 使用方法

根据文档中的提示，可以使用三种方式添加这6个事件的监听器

1.通过SpringApplication的addApplicationListener方法添加

```java
SpringApplication.run(SpringBootDocApplication.class,args).addApplicationListener(new SpringBootListener());
```

2.类似的用SpringApplicationBuilder实现

```java
new SpringApplicationBuilder(SpringBootDocApplication.class).listeners(new SpringBootListener()).run(args);
```

3.如果Listener很多，也可以写在配置文件中，在resources目录下新建META-INF/spring.factories

```java
org.springframework.context.ApplicationListener=com.ttyc.doc.extend.event.customer.SpringBootListener
```

最终项目一启动便输出: listening spring boot starting event
