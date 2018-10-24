---
title: Spring Boot 官方文档阅读
date: 2018-07-24 14:04:06
categories: Spring Boot
tags: [Spring Boot,官方文档阅读]
toc: true
comments: true
---

Spring Boot 版本 2.0.3.RELEASE

文档地址：[https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/html/](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/html/)

## 遇到的英文单词


* typical: 典型的
* transitively: 可传递地
* Several of : 几个
* dives into : 深入
* bootstrap : 引导
* delegate : 委托
* approach : 方法
* perform : 执行
* detect : 察觉，侦测，发现


## Spring CLI的使用
step 1： sdkman安装spring-boot

```bash
sdk install springboot
```

step 2：运行groovy脚本

```bash
spring run app.groovy
```

示例：

```java
@RestController
class ThisWillActuallyRun {

	@RequestMapping("/")
	String home() {
		"Hello World!"
	}

}
```

## 从低版本的Spring Boot升级到2.0

加入依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-properties-migrator</artifactId>
	<scope>runtime</scope>
</dependency>
```

运行一次后移除该依赖

## 使用maven命令启动Spring Boot

```bash
mvn spring-boot:run
```
相应的gradle有:
```bash
gradle bootRun
```

可以export系统变量(**没有测试**):

```bash
export MAVEN_OPTS=-Xmx1024m
```

## 社区提供的Spring Boot starter
[starters列表](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-project/spring-boot-starters)

## 如何排除不想生效的Bean
方式一：使用exclude属性
```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

方式二：如果classpath下没有这个类，使用类全名
```java
@SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")
```

方式三：如果有多个，可以使用spring.autoconfigure.exclude属性
```java
spring.autoconfigure.exclude=DataSourceAutoConfiguration.class
```

你可以同时在注解和属性上使用exclude

You can define exclusions both at the annotation level and by using the property.

## 构造器注入可以省略@Autowired

**If a bean has one constructor, you can omit the @Autowired**

```java
@Service
public class DatabaseAccountService implements AccountService {

	private final RiskAssessor riskAssessor;

	public DatabaseAccountService(RiskAssessor riskAssessor) {
		this.riskAssessor = riskAssessor;
	}
}
```

## 使用Remote Debug时的启动参数
仅供参考，自己还没有试

```bash
java -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8000,suspend=n 
-jar target/myapplication-0.0.1-SNAPSHOT.jar
```

## IDEA中使用devtools的正确姿势

修改代码后，需要点击: **Build->Build Project**

## 编程式的属性设置

```java
public static void main(String[] args) {
	System.setProperty("spring.devtools.restart.enabled", "false");
	SpringApplication.run(MyApp.class, args);
}
```

## Spring Boot提供的几个很有用的事件

针对Spring boot提供的事件，编写自己的Listener,详见**[Spring-ContextRefreshedEvent事件](https://greedypirate.github.io/2018/07/26/Spring-ContextRefreshedEvent%E4%BA%8B%E4%BB%B6/)**

1. ApplicationStartingEvent：在开始运行时，监听器注册和初始化之后被触发
2. ApplicationEnvironmentPreparedEvent：发现 Environment 被上下文使用时，上下文被创建之前触发
3. ApplicationPreparedEvent：在启动刷新之前，bean定义被加载之后被触发
4. ApplicationStartedEvent：上下文刷新之前，应用和命令行启动器运行之前触发
5. ApplicationReadyEvent：在所有应用和命令行启动器调用之后，这表明应用已经准备好处理请求
	6.ApplicationFailedEvent：启动时出现异常触发
	
## 配置文件的名称和位置

```java
spring.config.name
spring.config.location
```

**注**：这是两个需要很早初始化的属性，只能写在环境变量里(OS environment variable, a system property, or a command-line argument)

## 获取命令行参数

* 通过注入ApplicationArguments

```java
@Component
public class BootstrapArgs {

    @Autowired
    public BootstrapArgs(ApplicationArguments args) {
        boolean myargs = args.containsOption("myargs");
        Assert.state(myargs, "无法获取自定义参数");
        List<String> nonOptionArgs = args.getNonOptionArgs();
        System.out.println("nonOptionArgs : " + StringUtils.collectionToCommaDelimitedString(nonOptionArgs));
        Set<String> optionNames = args.getOptionNames();
        for (String optionName : optionNames) {
            List<String> optionValues = args.getOptionValues(optionName);
            System.out.println(optionValues);
        }
    }
}

```

* 实现CommandLineRunner或ApplicationRunner接口

```java
@Component
public class RunnerBean implements CommandLineRunner{
    @Override
    public void run(String... args) throws Exception {
        for(String arg:args){
            System.out.println(arg);
        }
    }
}
```

## 日志按环境生效
以下配置文件展示了多个环境，特定环境，非某个环境

```xml
<configuration>
    <!-- 测试环境+开发环境. 多个使用逗号隔开. -->
    <springProfile name="test,dev">
        <logger name="com.example.demo.controller" level="DEBUG" additivity="false">
            <appender-ref ref="consoleLog"/>
        </logger>
    </springProfile>

    <!-- 生产环境. -->
    <springProfile name="production">
    </springProfile>
    
    <!-- 非生产环境 -->
    <springProfile name="!production">
    ...
    </springProfile>
</configuration>
```

## 日志使用Spring环境变量

```xml
<springProperty scope="context" name="fluentHost" source="myapp.fluentd.host"
		defaultValue="localhost"/>
<appender name="FLUENT" class="ch.qos.logback.more.appenders.DataFluentAppender">
	<remoteHost>${fluentHost}</remoteHost>
	...
</appender>
```

## 项目首页查找规则

支持html和模板引擎作为首页，首先会查找index.html，然后查找index template，还是没有时，会默认用一个欢迎页

## WebBindingInitializer

用于配置全局的类型转换器, 局部的可以在Controller中使用@InitBinder标记在方法上(**百度所得**)

## Todo List
1. [x] 搭建jenkins，配合git自动构建发布Spring Boot项目
    1.1 么么哒
    1.2 哈哈
2. 研究编排系统Docker，K8s
3. 研究项目里的Spring扩展
4. 总结分布式链路追踪
5. mybatis官方文档, 源码, mybatis-plus, 通用mapper，代码生成
6. spring boot多数据源，分库分表，sharding-jdbc
 










