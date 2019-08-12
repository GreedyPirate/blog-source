---
title: Spring ContextRefreshedEvent事件
date: 2018-07-26 10:41:22
categories: Spring
tags: [Spring, Spring 扩展]
toc: true
comments: true
---


```
遇到的单词
infrastructure ： 基础设施
arbitrary : 任何的，所有的
```
## 介绍

本文主要在[观察者模式](https://greedypirate.github.io/2018/07/26/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/)的基础上研究Spring中的事件机制

ApplicationListener监听以下4个事件：ContextStartedEvent，ContextRefreshedEvent，ContextStartedEvent，ContextClosedEvent

实现对ApplicationContext刷新或初始化时的监听，测试中未出现加载两次的情况，如果需要加入`event.getApplicationContext().getParent()`判断

## 监听ContextRefreshedEvent 

```java
@Component
public class ContextEnvent implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        System.out.println("Spring Refreshed");
    }
}
```
自从Spring 4.2以后，可以使用@EventListener注解实现，相信用过Spring-Kafka的读者不会陌生这种写法

```java
@Component
public class AnnotateContextEvent{
    @EventListener
    public void handleRefresh(ContextRefreshedEvent event){
        System.out.println("Spring Refreshed by annotated approach");
    }
}
```

---
## 自定义事件

实现起来很简单，接下来尝试下Spring中的自定义事件

自定义事件需要继承ApplicationEvent，这个类并没有什么深意，只是简单封装EventObject加入了时间戳

### Step 1 : 定义事件——被观察者

```java
public class CustomerEvent extends ApplicationEvent{

    private String name;

    public CustomerEvent(Object source, String name) {
        super(source);
        this.name = name;
    }

    public String getName() {
        return name;
    }
}
```
### Step 2 : 定义监听器——观察者

```java
@Component
public class CustomerListener implements ApplicationListener<CustomerEvent>{
    @Override
    public void onApplicationEvent(CustomerEvent event) {
        System.out.println("CustomerListener listening： CustomerEvent has been triggered, event name is " + event.getName());
    }
}
```

### Step 3 : 多了一步事件发布

```java
@Component
public class CustomEventPublisher implements ApplicationEventPublisherAware {

    private ApplicationEventPublisher publisher;

    public void publish(){
        CustomerEvent customerEvent = new CustomerEvent(this,"click");
        publisher.publishEvent(customerEvent);
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }

}
```

## 测试

```java
@SpringBootApplication
public class SpringBootDocApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootDocApplication.class, args)
                .getBean(CustomEventPublisher.class).publish();
	}
}
```

## 控制台输出

```bash
CustomerListener listening： CustomerEvent has been triggered, event name is click
```

### 如果再加入一个监听者呢？是否能通知到

```java
@Component
public class ExtraListener implements ApplicationListener<CustomerEvent> {
    @Override
    public void onApplicationEvent(CustomerEvent event) {
        System.out.println("ExtraListener listening： CustomerEvent has been triggered, event name is " + event.getName());
    }
}
```

## 结果

```bash
CustomerListener listening： CustomerEvent has been triggered, event name is click
ExtraListener listening： CustomerEvent has been triggered, event name is click

```

--- 

## 源码跟踪

<img src="https://ae01.alicdn.com/kf/H01d163bde58c41e0b908f08d02d02af1N.png" width="65%" align="left"/>

这里和观察者模式的遍历一样，调用所有的监听器
<img src="https://ae01.alicdn.com/kf/H9c5679f3036b4c40b70c684e1d7cf4972.png" width="65%" align="left"/>

进入getApplicationListeners方法，可以看到如何查找注册在event上的Listener
<img src="https://ae01.alicdn.com/kf/Ha30a903295484252a75561e2fc2717b2f.png" width="65%" align="left"/>


根据@Order注解对Listener排序，
```java
AnnotationAwareOrderComparator.sort(allListeners);
```

对两个Listener加入@Order注解，果然值较小的ExtraListener先执行

注：@Order Lower values have higher priority


## 心得

	对自己的猜想要多验证


## 参考
 
[https://spring.io/blog/2015/02/11/better-application-events-in-spring-framework-4-2](https://spring.io/blog/2015/02/11/better-application-events-in-spring-framework-4-2)
[http://wiki.jikexueyuan.com/project/spring/custom-events-in-spring.html](http://wiki.jikexueyuan.com/project/spring/custom-events-in-spring.html)
[https://blog.csdn.net/tuzongxun/article/details/53637159](https://blog.csdn.net/tuzongxun/article/details/53637159)
[https://blog.csdn.net/zhangningzql/article/details/52515890](https://blog.csdn.net/zhangningzql/article/details/52515890)