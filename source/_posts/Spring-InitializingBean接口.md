---
title: Spring InitializingBean接口
date: 2018-07-25 14:17:12
categories: Spring
tags: [Spring,Spring 扩展]
toc: true
comments: true
---

## 源文档

InitializingBean接口的doc文档解释如下，大意为：

实现这个接口的bean，可以在BeanFactory设置完所有的属性之后生效，例如，执行自定义的bean初始化，或者只是为了检查所有的属性被设置了

另一个选择是指定`init-method`，例如在XML中

```
/**
 * Interface to be implemented by beans that need to react once all their
 * properties have been set by a BeanFactory: for example, to perform custom
 * initialization, or merely to check that all mandatory properties have been set.
 *
 * An alternative to implementing InitializingBean is specifying a custom
 * init-method, for example in an XML bean definition.
 */
```

## 测试代码

从接口描述上可以看出和指定*init-method*的作用应该是类似的,测试代码如下

### Step 1：实现InitializingBean接口

```java
public class InitBeanExtend implements InitializingBean{

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("after properties set");
    }

    public void init(){
        System.out.println("bean inited");
    }
}
```

### Step 2: 使用java config定义bean，指定init-method

为了方便指定init-method,使用java config

```java
@Configuration
public class InitConfig {
    
    @Bean(initMethod = "init")
    public InitBeanExtend initBeanExtend(){
        return new InitBeanExtend();
    }
}

```

### Step 3: 编写测试用例

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringBootDocApplicationTests {

	@Autowired
	InitBeanExtend initBeanExtend;

	@Test
	public void contextLoads() {
		InitBeanExtend bean = InitBeanExtend.class.cast(initBeanExtend);
	}

}
```

## 控制台输出

```bash
after properties set
bean inited
```
结果表明init-method是在afterPropertiesSet方法执行之后调用的

---

查看`AbstractAutowireCapableBeanFactory`类源码

```java
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
	...
	String initMethodName = mbd.getInitMethodName();
	final Method initMethod = (mbd.isNonPublicAccessAllowed() ?
				BeanUtils.findMethod(bean.getClass(), initMethodName) :
				ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));
	...
}
```

二者除了先后顺序的明显区别之外，可以看出init-method是通过反射达到目的的，而InitializingBean接口具有代码侵入性，有对Spring的依赖

注意: init-method方法不能有参数，否则将抛出异常

```
org.springframework.beans.factory.support.BeanDefinitionValidationException: Couldn't find an init method named 'init' on bean with name 'initBeanExtend'
```
在IDEA下会有编译警告
<img src="https://ae01.alicdn.com/kf/Hb3803affca7840a190a0a6a76efa6185L.png" width="65%" align="left"/>

## 实际应用

看过Spring源码的读者经常可以看到这个接口的使用，比如在bean初始化完属性之后，进行参数检查

## 扩展

### `DisposableBean`接口
与初始化相对应的还有销毁，在Spring中提供DisposableBean接口，可用来优雅的退出Spring Boot程序，对前面的代码添加实现
```java
public class InitBeanExtend implements InitializingBean,DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("after properties set");
    }

    public void init(){
        System.out.println("bean inited");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("gracefully close applicationContext");
    }
}
```

### Java EE5规范`@PostConstruct`和`@PreDestroy`

Java EE5规范提出的两个影响Servlet声明周期的方法，添加在非静态方法上，分别会在Servlet实例初始化之后和被销毁之前执行一次
