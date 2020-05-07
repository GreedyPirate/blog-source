---
title: Spring IOC容器启动之初始化上下文
date: 2020-02-03 18:48:57
categories: Spring
tags: [Spring,源码]
toc: true
comments: true
---

# 前言

本文采用AnnotationConfigApplicationContext作为上下文，如果是web容器大部分源码可用于参考，但不能保证一致。

Spring IOC容器的源码很多，分解为多篇文章来讲解，

# 测试代码
测试代码很简单，我这里用了spring自带的测试类ConfigForScanning和TestBean，大家也可以自己写一个

源码如下：

```java
@Configuration
public class ConfigForScanning {
   @Bean
   public TestBean testBean() {
      return new TestBean();
   }
}
```

以上方式也就是我们常说的java config，再写一个最常见的基于注解的上下文来测试

```java
public void testGetBean() {
   ApplicationContext context = new AnnotationConfigApplicationContext(ConfigForScanning.class);
   TestBean bean = context.getBean(TestBean.class);
}
```



# 初始化上下文

初始化上下文指的是创建AnnotationConfigApplicationContext对象，ConfigForScanning作为参数传入

源码如下：

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   this();
   register(componentClasses);
   refresh();
}
```

可以以看到只有三行代码，第一行调用了无参构造方法。

## 无参构造方法

我们知道java会在第一行隐式调用父类的无参构造方法

```java
public AnnotationConfigApplicationContext() {
   // super();
   this.reader = new AnnotatedBeanDefinitionReader(this);
   this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

AnnotationConfigApplicationContext的父类是GenericApplicationContext，大名鼎鼎的DefaultListableBeanFactory就在此创建了

```java
public GenericApplicationContext() {
   this.beanFactory = new DefaultListableBeanFactory();
}
```

此时回过头看AnnotationConfigApplicationContext，它初始化了AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner，我们可以理解为：

AnnotatedBeanDefinitionReader可以从@Configuration中读取BeanDefinition，而ClassPathBeanDefinitionScanner会从我们给定的classPath中扫描BeanDefinition

那么ClassPathBeanDefinitionScanner什么时候用到的呢，这里给大家一个小例子供参考

假设在org.springframework.context.annotation6包下有一个ComponentForScanning类，它被标注了@Component注解

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.scan("org.springframework.context.annotation6");
context.refresh();

ComponentForScanning bean = context.getBean(ComponentForScanning.class);
assertThat(bean).isNotNull();
```

至此，我们可以理解第一步是初始化了DefaultListableBeanFactory和2个工具类

## 注册bean的配置类

AnnotationConfigApplicationContext构造方法的第一行代码调用了无参构造方法，第二行代码则是把用户传进来的配置类解析为BeanDefinition，也就是ConfigForScanning类，该配置类会被上面的工具类AnnotatedBeanDefinitionReader所解析

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   this();
   // 把@Configuration标注的类，解析为AnnotatedGenericBeanDefinition，注册到BeanDefinitionRegistry
   register(componentClasses);
   refresh();
}

	public void register(Class<?>... componentClasses) {
		Assert.notEmpty(componentClasses, "At least one component class must be specified");
		this.reader.register(componentClasses);
	}
// 省略中间调用 ...
```

经过中间层层调用，来到doRegisterBean方法

### doRegisterBean

该方法是AnnotatedBeanDefinitionReader的注册配置类的具体实现，java config的bean用的是AnnotatedGenericBeanDefinition，它的类图如下

![Annotated-BeanDefinition](/Users/admin/mine/spring-pics/Annotated-BeanDefinition.png)

源码及注释如下，比较简单，不再细说

```java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
      @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
      @Nullable BeanDefinitionCustomizer[] customizers) {

   AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
   // 判断是否有Conditional注解，常见于Spring boot 
   if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
      return;
   }

   // 除了构造方法和工厂bean，还可以通过一个Supplier创建bean
   abd.setInstanceSupplier(supplier);
   // 解析scope信息，singleton
   abd.setScope(scopeMetadata.getScopeName());
   // 实现类是AnnotationBeanNameGenerator，获取beanName，实现过程看里面注释
   String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
   // 填充lazy,primary,dependOn,role,description等信息
   AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // 为null
   if (qualifiers != null) {
      for (Class<? extends Annotation> qualifier : qualifiers) {
         if (Primary.class == qualifier) {
            abd.setPrimary(true);
         }
         else if (Lazy.class == qualifier) {
            abd.setLazyInit(true);
         }
         else {
            abd.addQualifier(new AutowireCandidateQualifier(qualifier));
         }
      }
   }
   // 为null
   if (customizers != null) {
      for (BeanDefinitionCustomizer customizer : customizers) {
         customizer.customize(abd);
      }
   }

    // 注册到registry
   BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
   definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

至此AnnotationConfigApplicationContext初始化的第二步结束，接下来是第三步refresh方法，它是IOC容器初始化最核心的代码，下文会单独讲解

# 总结

AnnotationConfigApplicationContext的初始化前两步我们可以总结为以下几步
1. 创建DefaultListableBeanFactory
2. 初始化了用于扫描BeanDefinition的2个工具类：AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner
3. 将用户用@Configuration注解标注的bean配置类信息，注册到工厂中，等待初始化