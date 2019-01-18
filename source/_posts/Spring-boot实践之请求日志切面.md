---
title: Spring-boot实践之请求日志切面
date: 2019-01-02 17:08:10
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---

>记录请求日志切面的写法，和别人写的相比并无特殊之处

# 思路

## 日志信息
将controller中方法参数作为请求参数，返回值作为响应，这样做的前提是请求参数和返回值都已使用javabean封装，不一定适合每个人

## 耗时统计
tomcat为每个请求分配一个线程，自然想到使用ThreadLocal保存计时器，最后不要忘了remove

## HttpServletRequest对象的获取
直接注入即可，Spring底层也是用ThreadLocal实现的，具体实现参考RequestContextHolder

# 代码实现

```java
import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.util.StopWatch;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Aspect
@Slf4j
public class RequestLogAspect {

    private static final ThreadLocal<StopWatch> timer = new ThreadLocal<>();

    /**
     * 线程安全，可直接注入
     */
    @Autowired
    HttpServletRequest request;

    @Autowired
    HttpServletResponse response;

    @Pointcut("execution(* com.ttyc..controller.*(..))")
    public void controller() {
    }

    @Before("controller()")
    public void beforeRequest(JoinPoint joinPoint) {

        timer.get().start("requestTimeKeeperTask");

        try {
            log.info("request url: {}, \nparams are {}", request.getRequestURI(), JSON.toJSONString(joinPoint.getArgs()[0]));
        } catch (Exception e) {
            log.warn("转换请求参数时异常");
        }
    }

    @AfterReturning(value = "controller()", returning = "response")
    public void afterResponse(Object response) {
        long latency = timer.get().getTotalTimeMillis();

        try {
            log.info("response {}\nlatency is {}ms", JSON.toJSONString(response), latency);
        } catch (Exception e) {
            log.warn("转换结果时异常");
        }finally {
            timer.remove();
        }
    }

    @AfterThrowing(value = "controller()", throwing = "ex")
    public void afterThrows(Throwable ex) {
        long latency = timer.get().getTotalTimeMillis();

        log.warn("请求异常，错误信息为：{}\n 耗时{}ms", ex.getMessage(), latency);

        timer.remove();
    }
}
```