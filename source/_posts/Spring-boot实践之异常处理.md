---
title: Spring boot实践之异常处理
date: 2018-10-13 19:33:36
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---

# Spring boot实践之异常处理

在上一章[封装返回体]()中，已经对请求成功的情况进行了封装，接下来便是处理异常，服务的生产者需要通过状态码此次请求是否成功，出现异常时，错误信息是什么，形如:

```json
{
    "code": 1,
    "msg": "FAILED",
    "data": null
}
```

可以看出只需要`code`与`msg`, 参考 `org.springframework.http.HttpStatus`的实现，我们可以定义一个枚举来封装错误信息，对外暴露`getCode`，`getMsg`方法即可。由于异常属于一个基础模块，将这两个方法抽象到一个接口中。

错误接口

```java
public interface ExceptionEntity {

    Integer getCode();

    String getMsg();
}
```

以用户模块为例，所有用户相关的业务异常信息封装到`UserError`中，例如用户不存在，密码错误

```java
public enum UserError implements ExceptionEntity {

    NO_SUCH_USER(1, "用户不存在"),
    ERROR_PASSWORD(2, "密码错误"),
    ;

    private final Integer MODULE = 10000;

    private Integer code;

    private String msg;

    UserError(Integer code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    @Override
    public Integer getCode() {
        return MODULE + this.code;
    }

    @Override
    public String getMsg() {
        return this.msg;
    }
}

```

需要注意的地方是笔者定义了一个`MODULE`字段，10000代表用户微服务，这样在拿到错误信息之后，可以很快定位报错的应用

自定义异常

```java
@Data
// lombok自动生成构造方法
@AllArgsConstructor
public class ServiceException extends RuntimeException{
    ExceptionEntity error;  
}
```

需要说明的是错误接口与自定义异常属于公共模块，而`UserError`属于用户服务

之后，便可以抛出异常

```java
throw new ServiceException(UserError.ERROR_PASSWORD);
```

目前来看，我们只是较为优雅的封装了异常，此时请求接口返回的仍然是Spring boot默认的错误体，没有错误信息

```java
{
    "timestamp": "2018-10-18T12:28:59.150+0000",
    "status": 500,
    "error": "Internal Server Error",
    "message": "No message available",
    "path": "/user/error"
}
```

接下来的异常拦截方式，各路神仙都有自己的方法，笔者只说Spring boot项目中比较通用的`@ControllerAdvice`，由于是Restful接口，这里使用`@RestControllerAdvice`

```java
// 注意这属于基础模块，扫描路径不要包含具体的模块，用..代替
@RestControllerAdvice(basePackages="com.ttyc..controller",annotations={RestController.class})
// lombok的日志简写
@Slf4j
public class ControllerExceptionAdvisor{

    @ExceptionHandler({ServiceException.class})
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseModel handleServiceException(ServiceException ex){
        Integer code = ex.getError().getCode();
        String msg = ex.getError().getMsg();
        log.error(msg);

        ResponseModel model = new ResponseModel();
        model.setCode(code);
        model.setMsg(msg);

        return model;
    }
    
    /**
     * 其他错误
     * @param ex
     * @return
     */
    @ExceptionHandler({Exception.class})
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseModel exception(Exception ex) {
        int code = HttpStatus.INTERNAL_SERVER_ERROR.value();
        String msg = HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase();
        log.error(msg);

        ResponseModel model = new ResponseModel();
        model.setCode(code);
        model.setMsg(msg);

        return model;
    }
}
```

具有争议的一点是捕获`ServiceExcption`之后，应该返回200还是500的响应码，有的公司返回200，使用`code`字段判断成功失败，这完全没有问题，但是按照Restful的开发风格，这里的`@ResponseStatus`笔者返回了500，请读者根据自身情况返回响应码

### 测试接口与测试用例 

#### 测试接口

```java
    @GetMapping("error")
    public boolean error(){
        // 抛出业务异常示例
        throw new ServiceException(UserError.NO_SUCH_USER);
    }
```



#### 测试用例

```java
    @Test
    public void testError() throws Exception {
        String result =
                mockMvc.perform(get("/user/error"))
                        .andExpect(status().isInternalServerError())
                        .andReturn().getResponse().getContentAsString();
        System.out.println(result);
    }
```

结果为:

```json
{
	"data": null,
	"code": 10001,
	"msg": "用户不存在"
}
```
