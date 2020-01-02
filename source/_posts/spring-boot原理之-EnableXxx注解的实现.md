---
title: spring-boot原理之@EnableXxx注解的实现
date: 2019-09-25 11:59:12
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---
>Spring boot各种领人眼花缭乱的starter层出不穷，它实现了各种组件与spring的集成，本文以spring-cloud-openfeign 2.2.0版本为例，介绍@EnableXxx注解的实现原理

# 准备
*注 由于有包扫描相关，本文约定包名为com.ttyc，启动类名为MainApplication*

@EnableFeignClients的源码大致如下
```java
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
	// value和basePackages的作用一样，互为别名
	String[] value() default {};
	String[] basePackages() default {};

	Class<?>[] basePackageClasses() default {};
	Class<?>[] defaultConfiguration() default {};
	Class<?>[] clients() default {};
}
```
它引入了FeignClientsRegistrar，而FeignClientsRegistrar实现了ImportBeanDefinitionRegistrar接口

## @Import注解
@Import注解通常用于导入@Configuration注解的配置类，但在它的文档描述中也明确说明了支持ImportSelector和ImportBeanDefinitionRegistrar的实现类
```
 * <p>Provides functionality equivalent to the {@code <import/>} element in Spring XML.
 * Allows for importing {@code @Configuration} classes, {@link ImportSelector} and
 * {@link ImportBeanDefinitionRegistrar} implementations
```
ImportBeanDefinitionRegistrar接口提供import类的注解元信息，下文将会解释它是什么，以及一个BeanDefinition的注册表用于注册
```java
	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	}

```
# 正文

## FeignClientsRegistrar
FeignClientsRegistrar实现的registerBeanDefinitions调用了2个方法
```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}
}
```

### registerDefaultConfiguration

registerDefaultConfiguration注册了一个FeignClientSpecification的bean，它的beanName经过一系列字符串拼接，最终是default.com.ttyc.MainApplication
用于Feign的同学都知道Feign可以自定义一些配置，如Decoder，Encoder，Contract，这里是注册了一个默认的配置
```java
private void registerDefaultConfiguration(AnnotationMetadata metadata,
		BeanDefinitionRegistry registry) {
	// 拿到EnableFeignClients注解的配置项
	Map<String, Object> defaultAttrs = metadata
			.getAnnotationAttributes(EnableFeignClients.class.getName(), true);

	// 肯定包含defaultConfiguration
	if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
		String name;
		// 有没有标注在内部类上
		if (metadata.hasEnclosingClass()) {
			name = "default." + metadata.getEnclosingClassName();
		}
		else {
			// 基本都走这 name = default.com.ttyc.MainApplication
			name = "default." + metadata.getClassName();
		}
		// 注册bean
		registerClientConfiguration(registry, name,
				defaultAttrs.get("defaultConfiguration"));
	}
}
```

#### AnnotationMetadata
至此可以看出AnnotationMetadata其实就是EnableFeignClients注解所在类的元信息，通过AnnotationMetadata的getAnnotationAttributes方法可以很方便的获到一个注解所有属性和值的map。但EnableFeignClients注解不是随便什么类都可以写的，通常标注在启动类上也是原因的。

### registerFeignClients
我们写的FeignClient接口由ClassPathScanningCandidateComponentProvider负责扫描，它需要指定路径，以及过滤器
过滤器的作用是在扫描是获取匹配的类，在这里就是有FeignClient注解的类，最终组装成BeanDefinition集合
registerFeignClients的源码主要分为：获取basePackages，扫码到所有类封装为BeanDefinition，注册到spring IOC容器中

#### 获取basePackages流程
这一部分代码主要是希望大家不要随意设置clients属性，它会获取clients数组里每一个类所在的包，添加到basePackages集合中，实际开发中维护性并不是很好
那么在没有设置clients属性时，执行basePackages = getBasePackages(metadata)，它会依次添加用户在@EnableFeignClients中设置的value，basePackages以及basePackageClasses中每一个类所在的包路径，
如果这三个属性你都没有设置，就获取@EnableFeignClients注解所在类的包路径作为basePackages
```java
public void registerFeignClients(AnnotationMetadata metadata,
		BeanDefinitionRegistry registry) {
	ClassPathScanningCandidateComponentProvider scanner = getScanner();
	scanner.setResourceLoader(this.resourceLoader);

	Set<String> basePackages;

	Map<String, Object> attrs = metadata
			.getAnnotationAttributes(EnableFeignClients.class.getName());
	AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
			FeignClient.class);
	final Class<?>[] clients = attrs == null ? null
			: (Class<?>[]) attrs.get("clients");
	if (clients == null || clients.length == 0) {
		scanner.addIncludeFilter(annotationTypeFilter);
		basePackages = getBasePackages(metadata);
	}
	else {
		// @EnableFeignClients可以设置clients属性，如果设置了FeignClient类，就以它所在包为路径扫描
		final Set<String> clientClasses = new HashSet<>();
		basePackages = new HashSet<>();
		for (Class<?> clazz : clients) {
			basePackages.add(ClassUtils.getPackageName(clazz));
			clientClasses.add(clazz.getCanonicalName());
		}
		AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
			@Override
			protected boolean match(ClassMetadata metadata) {
				String cleaned = metadata.getClassName().replaceAll("\\$", ".");
				return clientClasses.contains(cleaned);
			}
		};
		scanner.addIncludeFilter(
				new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
	}
	// 见下文
}
```
### 注册FeignClient到IOC容器
在获取到所有的basePackages后，对其进行遍历，scanner扫描每一个路径，获取所有带有@FeignClient的类，并解析为BeanDefinition集合，
第二个for循环对BeanDefinition集合遍历，通过registerFeignClient方法注册到IOC容器中
而registerClientConfiguration是为每一个FeignClient的configuration注册bean，它的beanName为：@FeignClient的name值 + .FeignClientSpecification
```java
for (String basePackage : basePackages) {
	// 扫描包下所有类，把满足TypeFilter的类解析为BeanDefinition返回
	// 通常情况下scanner只addIncludeFilter一个annotationTypeFilter
	// TypeFilter: AnnotationTypeFilter过滤有FeignClient注解的类
	Set<BeanDefinition> candidateComponents = scanner
			.findCandidateComponents(basePackage);
	for (BeanDefinition candidateComponent : candidateComponents) {
		if (candidateComponent instanceof AnnotatedBeanDefinition) {
			// verify annotated class is an interface
			AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
			// 获取FeignClient类上的所有注解及其配置项，除了@FeignClient，还会有别的注解
			AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
			Assert.isTrue(annotationMetadata.isInterface(),
					"@FeignClient can only be specified on an interface");

			// 单独获取@FeignClient的配置项
			Map<String, Object> attributes = annotationMetadata
					.getAnnotationAttributes(
							FeignClient.class.getCanonicalName());

			// 从attributes中获取@FeignClient的name
			String name = getClientName(attributes);
			registerClientConfiguration(registry, name,
					attributes.get("configuration"));

			registerFeignClient(registry, annotationMetadata, attributes);
		}
	}
}
```

#### registerFeignClient
registerFeignClient方法就比较简单了，就是组装BeanDefinition，然后注册成一个FeignClientFactoryBean，它的beanName为FeignClient的类全名，里面还有一些别名，primary的设置，内部细节见源码里的注释

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
		AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
	String className = annotationMetadata.getClassName();
	BeanDefinitionBuilder definition = BeanDefinitionBuilder
			.genericBeanDefinition(FeignClientFactoryBean.class);
	validate(attributes);
	definition.addPropertyValue("url", getUrl(attributes));
	definition.addPropertyValue("path", getPath(attributes));
	String name = getName(attributes);
	definition.addPropertyValue("name", name);
	// contextId新版本加入的，用于自定义bean的别名，以前总是用name作为别名
	String contextId = getContextId(attributes);
	definition.addPropertyValue("contextId", contextId);
	definition.addPropertyValue("type", className);
	definition.addPropertyValue("decode404", attributes.get("decode404"));
	definition.addPropertyValue("fallback", attributes.get("fallback"));
	definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
	definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

	// 设置bean的别名
	String alias = contextId + "FeignClient";
	AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

	boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
															// null

	beanDefinition.setPrimary(primary);

	String qualifier = getQualifier(attributes);
	if (StringUtils.hasText(qualifier)) {
		// qualifier属性也是新加的，它会覆盖contextId的别名
		alias = qualifier;
	}

	// 类全名来作为beanName来注册bean
	BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
			new String[] { alias });
	BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

# 总结
@EnableFeignClients依赖了@Import可以导入ImportBeanDefinitionRegistrar实现类的特性，将FeignClient的接口类，配置注册到spring容器中，而在注册之前，需要对用户配置的一系列包扫描路径解析，获取到FeignClient的BeanDefinition，最终完成注册
FeignClientSpecification表示每一个FeignClient对应的配置，FeignClientFactoryBean表示每一个FeignClient

















































