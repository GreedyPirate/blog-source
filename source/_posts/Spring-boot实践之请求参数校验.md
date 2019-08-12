---
title: Spring boot实践之请求参数校验
date: 2018-10-14 15:07:44
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---

# Spring boot实践之请求参数校验

>本文讲述的是后端参数校验，在实际开发中，参数校验是前后端都要做的工作，因为请求接口的人除了普通用户，还有有各路神仙。

## 常规校验的痛楚

通常的校验代码如下

```java
@PostMapping("login")
public User login(@RequestBody User user){
    if(StringUtils.isBlank(user.getUsername())){
    	throw new RuntimeException("请输入用户名");
    }
    return user;
}
```

如果还有n个接口需要校验username，你可能会抽取`if`语句到一个方法中，过段时间你又会发现，不光要校验username，还要password，adress等等一堆字段，总结起来

1. 重复劳动
2. 代码冗长，不利于阅读业务逻辑
3. 出现问题要去不同的接口中查看校验逻辑

这无疑是件让人崩溃的事情，此时作为一个开发人员，你已经意识到需要一个小而美的工具来解决这个问题，你可以去google，去github搜索这类项目，而不是毫无作为，抑或者是自己去造轮子

## JSR303

JSR303规范应运而生，其中比较出名的实现就是Hibernate Validator，已包含在`spring-boot-starter-web`其中,不需要重新引入，`javax.validation.constraints`包下常用的注解有

| 注解                           | 含义                                                         |
| :----------------------------- | :----------------------------------------------------------- |
| @NotNUll                       | 值不能为空                                                   |
| @Null                          | 值必须为空                                                   |
| @Pattern(regex=)               | 值必须匹配正则表达式                                         |
| @Size(min=,max=)               | 集合的大小必须在min~max之间，如List，数组                    |
| @Length(min=,max=)             | 字符串长度                                                   |
| @Range(min,max)                | 数字的区间范围                                               |
| @NotBlank                      | 字符串必须有字符                                             |
| @NotEmpty                      | 集合必须有元素，字符串                                       |
| @Email                         | 字符串必须是邮箱                                             |
| @URL                           | 字符串必须是url                                              |
| @AssertFalse                   | 值必须是false                                                |
| @AssertTrue                    | 值必须是true                                                 |
| @DecimalMax(value=,inclusive=) | 值必须小于等于(inclusive=true)/小于(inclusive=false) value属性指定的值。可以注解在字符串类型的属性上 |
| @DecimalMin(value=,inclusive=) | 值必须大于等于(inclusive=true)/大f (inclusive=false) value属性指定的值。可以注解在字符串类型的属性上 |
| @Digits(integer-,fraction=)    | 数字格式检查。integer指定整 数部分的最大长度，fraction指定小数部分的最大长度 |
| @Future                        | 值必须是未来的日期                                           |
| @Past                          | 值必须是过去的日期                                           |
| @Max(value=)                   | 值必须小于等于value指定的值。不能注解在字符串类型的属性上    |
| @Min(value=)                   | 值必须大于等于value指定的值。不能注解在字符串类型的属性上    |
| ...                            | ...                                                          |


## 校验实战

接下来我们尝试一个入门例子,有一个User java bean, 为username字段加入@NotBlank注解，注意@NotBlank的包名

```java
import lombok.Data;

import javax.validation.constraints.NotBlank;

@Data
public class User {

    private Long id;

    @NotBlank(message = "请输入用户名")
    private String username;

    private String password;
}

```

表明将对username字段做非null，非空字符串校验，并为user参数添加@Valid

```java
public User login(@RequestBody @Valid User user)
```



按照[Spring boot实践之编写接口测试用例]()编写一个测试用例

```java
@Test
public void testBlankName() throws Exception {
    String params = "{\"id\": 101,\"username\": \"\",\"password\": \"1234\"}";
    mockMvc.perform(post("/user/login")
    .contentType(MediaType.APPLICATION_JSON_UTF8)
    .content(params))
    .andExpect(status().isBadRequest());
}
```

由于参数为空，将返回BadRequest—400响应码，但是此时我们获取不到错误信息，由于spring的拦截，甚至你会发现不进方法断点，仅仅得到一个400响应码，对前端提示错误信息帮助不大，因此我们需要获取错误信息

## 获取错误信息

```java
    @PostMapping("login")
    public User login(@Valid @RequestBody User user, BindingResult result){
        if(result.hasErrors()){
            result.getFieldErrors().stream().forEach(error -> {
                // ....
            });
        }
        return user;
    }
```

此时我们发现已经进入方法断点

![进入断点](https://ae01.alicdn.com/kf/H954c71c251af4bbbb3b337fb595cdaa5c.png)

## 统一异常处理

继续优化，想必大家也发现了，难道每个方法都要写`if`? 当然不用，ControllerAdvice不就是专门封装错误信息的吗，仿照[异常处理]()中的处理方式，我们很容易写出以下代码

```java
@ExceptionHandler({MethodArgumentNotValidException.class})
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ResponseModel exception(MethodArgumentNotValidException ex) {
    ResponseModel model = new ResponseModel();
    model.setCode(HttpStatus.BAD_REQUEST.value());
    model.setMsg(buildErrorMessage(ex));
    return model;
}

/**
 * 构建错误信息
 * @param ex
 * @return
 */
private String buildErrorMessage(MethodArgumentNotValidException ex){
    List<ObjectError> objectErrors = ex.getBindingResult().getAllErrors();
    StringBuilder messageBuilder = new StringBuilder();
    objectErrors.stream().forEach(error -> {
        if(error instanceof FieldError){
            FieldError fieldError = (FieldError) error;
            messageBuilder.append(fieldError.getDefaultMessage()).append(",");
        }
    });
    String message  = messageBuilder.deleteCharAt(messageBuilder.length() - 1).toString();
    log.error(message);
    return message;
}
```

### 注意点

除了使用`@ExceptionHandler`来捕获`MethodArgumentNotValidException`以外，还可以覆盖`ResponseEntityExceptionHandler`抽象类的handleMethodArgumentNotValid方法，但是二者不可以混用


## 自定义校验规则

由于JSR303提供的注解有限，实际开发过程中校验往往需要结合实际需求，JSR303提供了自定义校验扩展接口

典型的一个请求场景是枚举类型参数，假设用户分为3类: 普通用户，VIP玩家，氪金玩家，分别用1，2，3表示，此时如何校验前端传入的值在范围内，抖机灵的朋友可能会想到@Range，万一是离散的不连续数呢？

自定义注解类

```java
import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.*;

@Documented
// 指定校验类
@Constraint(validatedBy = InValidator.class)
@Target( { ElementType.METHOD, ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface In {
    String message() default "必须在允许的数值内";

    int[] values();

    // 用于分组校验
    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

注解的校验器

```java
import com.google.common.collect.Sets;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import java.util.Set;

public class InValidator implements ConstraintValidator<In, Number> {// 校验Number类型 
	
    private Set<Integer> inValues;

    @Override
    public void initialize(In in) { 
    	inValues = Sets.newHashSet();
    	int[] arr = in.values();
    	for(int a : arr){
    	   inValues.add(a);
    	}
    }

    @Override
    public boolean isValid(Number propertyValue, ConstraintValidatorContext cxt) {
        if(propertyValue==null) {
            return false;
        }
       return inValues.contains(propertyValue.intValue());
    }
}

```

至此，生产级别的参数校验基本完成

## 分组校验

在不同接口中，指定不同的校验规则，如：

1. 不同的接口，校验不同的字段
2. 同一个字段，在不同的接口中有不同的校验规则

以下实现第一种情况



首先定义两个空接口，代表不同的分组，也就是不同的业务

```java
public interface NewUser { }
public interface RMBUser { }
```

在指定校验规则时，指定分组

```java
public class User {
	// 省略...
    @NotBlank(groups = {NewUser.class}, message = "请输入密码")   
    private String password;

    @In(groups = {RMBUser.class}, values = {1,2,3}, message = "非法的用户类型")
    private Integer type;
}

```

### 不同的接口指定不同的校验分组

```java
// 省略类定义...
@PostMapping("normal")
public User normal(@Validated({NewUser.class}) @RequestBody User user){
    return user;
}

@PostMapping("rmb")
public User rmb(@Validated({RMBUser.class}) @RequestBody User user){
    return user;
}
```

编写测试用例

只检验密码

```java
	@Test
    public void testNormal() throws Exception {
        String params = "{\"id\": 101,\"username\": \"tom\",\"password\": \"\",\"type\": \"5\"}";
        String result = mockMvc.perform(post("/user/normal")
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .content(params))
                .andExpect(status().isBadRequest())
                .andReturn().getResponse().getContentAsString();
        System.out.println(result);
    }
```

输出：`{"data":null,"code":400,"msg":"请输入密码"}`
只检验用户类型

```java
	@Test
    public void testRMB() throws Exception {
        String params = "{\"id\": 101,\"username\": \"tom\",\"password\": \"\",\"type\": \"5\"}";
        String result = mockMvc.perform(post("/user/rmb")
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .content(params))
                .andExpect(status().isBadRequest())
                .andReturn().getResponse().getContentAsString();
        System.out.println(result);
    }
```

输出：`{"data":null,"code":400,"msg":"非法的用户类型"}`
