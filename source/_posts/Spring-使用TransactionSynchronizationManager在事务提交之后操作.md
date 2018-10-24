---
title: Spring 使用TransactionSynchronizationManager在事务提交之后操作
date: 2018-08-04 19:55:40
categories: Spring
tags: [Spring,事务,异步]
toc: true
comments: true
---

需求: 在事务执行成功后，再去进行某一项操作,如发送邮件、短信等，此类事件通常是非及时触发的，所以采用异步执行

## TransactionSynchronizationManager

使用TransactionSynchronizationManager注册一个事务同步适配器，以下代码会在插入一个User对象后输出一段文字，

```java
@Transactional
public Boolean regist() {
    User jack = User.builder().name("jack").phone("110").build();
    userDao.save(jack);

    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        @Override
        public int getOrder() {
            return 1;
        }

        @Override
        public void afterCommit() {
            System.out.println("send a mail");
        }

    });
    if (true) throw new RuntimeException("fail insert");
    return Boolean.FALSE;
}
```

getOrder方法的作用:*They will be executed in an order according to their order value (if any)*

注册的适配器按照顺序执行，例如我可以先发邮件，再发短信，但在异步机制中无法保证有序



`TransactionSynchronization`接口中还提供了以下方法

* beforeCommit：事务提交之前，在`beforeCompletion`之前
* beforeCompletion: 事务提交/回滚之前
* afterCompletion: 事务提交/回滚之后

## 异步执行
初始化一个线程池

```java
@Configuration
@EnableAsync
public class TaskPoolConfig {
    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(200);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("taskExecutor-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}
```
编写异步任务代码

```
@Service
public class NotifyService {
    @Async
    public void sendMail(String message){
        System.out.println(message);
    }
}
```
在afterCommit中调用即可

参考：

1. [https://segmentfault.com/a/1190000004235193](https://segmentfault.com/a/1190000004235193)
2. [http://azagorneanu.blogspot.com/2013/06/transaction-synchronization-callbacks.html](http://azagorneanu.blogspot.com/2013/06/transaction-synchronization-callbacks.html)

