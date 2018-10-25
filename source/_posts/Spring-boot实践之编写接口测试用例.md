---
title: Spring boot实践之编写接口测试用例
date: 2018-10-13 14:47:12
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---
# Spring boot实践之编写接口测试用例

>  测试用例对开发者降低bug率,方便测试人员回归测试有十分重要的意义。



本文介绍如何使用`MockMvc`编写测试用例. 

在Spring boot项目中编写测试用例十分简单，通常建立一个Spring boot项目都会test目录下生成一个Test类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class DemoApplicationTests {
    @Test
    public void contextLoads() throws Exception {
    }
}
```

以用户查询为例，通常有一个用户实体，以及`UserController`

```java
// @Data注解来自lombok
@Data
public class User {

    private Long id;

    private String username;

    private String password;
}
```

getInfo方法是一个restful接口，模拟查询用户详情

```java
@RestController
@RequestMapping("user")
public class UserController {

    @GetMapping("info")
    public User getInfo(@RequestParam(name = "name", required = true) String username){
        User user = new User();
        user.setUsername(username + "s");
        return user;
    }
}
```



以下通过MockMvc对象，测试`/user/info}`请求是否成功，并符合预期

```java
import com.alibaba.fastjson.JSONObject;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SecurityDemoApplicationTests {

    //注入上下文对象
    @Autowired
    private WebApplicationContext context;

    private MockMvc mockMvc;

    @Before
    public void setup() {
        //初始化mockMvc对象
        mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
    }

    @Test
    public void testQuery() throws Exception {
        String result =
                //执行get请求，这里有个小坑，第一个/必须有
                mockMvc.perform(get("/user/info")
                        //设置content-type请求头
                        .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                        //设置参数  
                        .param("name", "jay"))
                        //预期的相应码是200-ok
                        .andExpect(status().isOk())
                        //预测username的值为jays
                        .andExpect(jsonPath("$.username").value("jays"))
                        //获取响应体
                        .andReturn().getResponse().getContentAsString();
        System.out.println(result);
    }
}
```

最终通过测试，并输出响应体

```json
{
    "id": 101,
    "username": "jays",
    "password": "1234"
}
```

关于`$.id`jsonpath的使用，参考[JsonPath](https://github.com/json-path/JsonPath)

同时付一段使用json参数的post请求方式，大同小异，

```java
String params = "{\"id\": 101,\"username\": \"jason\",\"password\": \"1234\"}";
mockMvc.perform(post("/user/login")
        .contentType(MediaType.APPLICATION_JSON_UTF8)
        .content(params))
        .andExpect(status().isOk());
```

注意后端接受json格式参数的方式：`方法名(@RequestBody User user)` 