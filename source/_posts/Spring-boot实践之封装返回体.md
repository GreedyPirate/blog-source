---
title: Spring boot实践之封装返回体
date: 2018-10-11 17:51:05
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---

# Spring boot实践之封装返回体

在实际开发中，一个项目会形成一套统一的返回体接口规范，常见的结构如下

```json
{
    "code": 0,
    "msg": "SUCCESS",
    "data": 真正的数据
}
```

读者可以根据自己的实际情况封装一个java bean，刑如：

```java
@Data
public class ResponseModel<T> {
    private T data;
    private Integer code;
    private String msg;
}
```

在spring boot中，会将返回的实体类，通过jackson自动转换成json

Spring提供了`ResponseBodyAdvice`接口拦截响应体

```java
public class ResponseAdvisor implements ResponseBodyAdvice {
    @Override
    public boolean supports(MethodParameter methodParameter, Class aClass) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body,
                                  MethodParameter methodParameter, 
                                  MediaType mediaType,
                                  Class aClass, 
                                  ServerHttpRequest serverHttpRequest, 
                                  ServerHttpResponse serverHttpResponse) {
        ResponseModel model = new ResponseModel();
        model.setCode(0);
        model.setData(body);
        model.setMsg("SUCCESS");
        return model;
    }
}

```

这只是一个最初的功能，值得优化的地方有很多，读者应根据自己的情况进行扩展

根据笔者遇到的情况，抛砖引玉一下

1. 是否需要对所有的响应拦截，可以在supports方法中判断
2. 下载返回的是字节数据，再进行包装必然得不到正确的文件，不应该进行包装