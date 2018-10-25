---
title: Spring Boot 内容协商
date: 2018-08-23 16:08:13
categories: Spring Boot
tags: [Spring,Spring Boot,Spring 扩展]
toc: true
comments: true
---

新建实体

```java
import lombok.Builder;
import lombok.Data;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlRootElement;

@Data
@Builder
@XmlRootElement
@XmlAccessorType(XmlAccessType.FIELD)
public class User {
    @XmlElement
    private Long id;

    @XmlElement
    private String name;

    @XmlElement
    private String phone;
}
```

配置

`parameter-name: type`表示参数名

```yaml
spring:
  mvc:
    message-codes-resolver-format: PREFIX_ERROR_CODE
    contentnegotiation:
      favor-parameter: true
      parameter-name: type
      media-types:
        json: application/json
        xml: application/xml
```

接口

```java
@RestController
public class ContentNegotiationController {

    @PostMapping("user")
    public User user(){
        User user = User.builder().id(1L).name("jay").phone("183").build();
        return user;
    }
}
```

测试接口













