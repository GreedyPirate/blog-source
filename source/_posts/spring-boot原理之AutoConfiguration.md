---
title: spring-boot原理之AutoConfiguration
date: 2019-11-27 21:05:38
tags:
	- Spring Boot
toc: true
comments: true
---
>承接上篇，本文分析@Import的另一种引入方式，通过实现ImportSelector的方式来引入bean

# 准备
@SpringBootApplication的源码如下，它是一个组合注解的事实想必大家已经知道，其中@EnableAutoConfiguration实现了主要的自动配置逻辑
```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};
}
```

@EnableAutoConfiguration注解又使用了熟悉的@Import注解，它的作用在[spring-boot原理之@EnableXxx注解的实现]()中已经说过了，它是可以通过ImportSelector引入bean的
```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```
## ImportSelector

ImportSelector的逻辑很简单，你告诉它要引入bean的类名即可
```java
	/**
	 * Select and return the names of which class(es) should be imported based on
	 * the {@link AnnotationMetadata} of the importing @{@link Configuration} class.
	 */
	String[] selectImports(AnnotationMetadata importingClassMetadata);
```

## META-INF目录下的两个文件

spring.factories: 
spring-autoconfigure-metadata.properties
