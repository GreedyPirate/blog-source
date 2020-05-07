---
title: Spring IOC容器之解析并注册BeanDefinition
date: 2020-02-08 20:44:51
categories: Spring
tags: [Spring,源码]
toc: true
comments: true
---

# 前言

ConfigurationClassPostProcessor作为当前唯一的BeanFactoryPostProcessor，而且还是一个特殊的BeanDefinitionRegistryPostProcessor(继承自BeanFactoryPostProcessor)。

按前文所说，先执行的是postProcessBeanDefinitionRegistry方法，后执行postProcessBeanFactory

# BeanDefinitionRegistry后置处理

抛去一些判断不说，该方法主要是调用了processConfigBeanDefinitions

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
   int registryId = System.identityHashCode(registry);
   if (this.registriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
   }
   if (this.factoriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
   }
   this.registriesPostProcessed.add(registryId);

   processConfigBeanDefinitions(registry);
}
```

## processConfigBeanDefinitions

该方法主要是从各个地方解析BeanDefinition，并注册到BeanDefinitionRegistry，而DefaultListableBeanFactory继承自BeanDefinitionRegistry，我们也可以理解为注册到Spring IOC容器中

首先我们要知道Spring中有哪些可以注册Bean的方式，以下进行例举

1. xml配置文件，最原始的方式。groovy等不在讨论范围之内
2. 通过java config，即@Configuration+@Bean结合的方式
3. @Component，@Service….等注解
4. 通过@ComponentScan，@ComponentScans扫描
5. 通过@Import引入配置类，它又可细分为3类：@Configuration，ImportSelector，ImportBeanDefinitionRegistrar
6. 通过@ImportResource引入配置文件

此处我们仅讨论java config的方式，spring的做法主要分为以下几步

1. 每个被@Configuration标注的类抽象成一个ConfigurationClass对象
2.  将@Configuration注解的类中所有@Bean方法，解析成一个BeanMethod，然后添加到ConfigurationClass的beanMethods属性中
3. 由于后续需要AOP增强，需要验证是否满足CGLIB的限制，比如类和方法不能是final，方法不能是private等
4. 将ConfigurationClass中的beanMethods解析成BeanDefinition并注册

前2步由ConfigurationClassParser的parse方法完成，第3步由validate方法完成，最后一步是由ConfigurationClassBeanDefinitionReader的loadBeanDefinitions方法完成

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
   List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
   String[] candidateNames = registry.getBeanDefinitionNames();

   for (String beanName : candidateNames) {
      BeanDefinition beanDef = registry.getBeanDefinition(beanName);
      if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
         if (logger.isDebugEnabled()) {
            logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
         }
      }
      else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
         configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
      }
   }

   // Return immediately if no @Configuration classes were found
   if (configCandidates.isEmpty()) {
      return;
   }

   // Sort by previously determined @Order value, if applicable
   configCandidates.sort((bd1, bd2) -> {
      int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
      int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
      return Integer.compare(i1, i2);
   });

   // Detect any custom bean name generation strategy supplied through the enclosing application context
   SingletonBeanRegistry sbr = null;
   if (registry instanceof SingletonBeanRegistry) {
      sbr = (SingletonBeanRegistry) registry;
      if (!this.localBeanNameGeneratorSet) {
         BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
               AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
         if (generator != null) {
            this.componentScanBeanNameGenerator = generator;
            this.importBeanNameGenerator = generator;
         }
      }
   }

   if (this.environment == null) {
      this.environment = new StandardEnvironment();
   }

   // Parse each @Configuration class
   ConfigurationClassParser parser = new ConfigurationClassParser(
         this.metadataReaderFactory, this.problemReporter, this.environment,
         this.resourceLoader, this.componentScanBeanNameGenerator, registry);

   Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
   Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
   do {
      parser.parse(candidates);
      parser.validate();

      // 是从ConfigurationClassParser中获取解析的结果：Set<ConfigurationClass>
      Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
      configClasses.removeAll(alreadyParsed);

      // Read the model and create bean definitions based on its content
      if (this.reader == null) {
         this.reader = new ConfigurationClassBeanDefinitionReader(
               registry, this.sourceExtractor, this.resourceLoader, this.environment,
               this.importBeanNameGenerator, parser.getImportRegistry());
      }
      this.reader.loadBeanDefinitions(configClasses);
      alreadyParsed.addAll(configClasses);

      candidates.clear();
      if (registry.getBeanDefinitionCount() > candidateNames.length) {
         String[] newCandidateNames = registry.getBeanDefinitionNames();
         Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
         Set<String> alreadyParsedClasses = new HashSet<>();
         for (ConfigurationClass configurationClass : alreadyParsed) {
            alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
         }
         for (String candidateName : newCandidateNames) {
            if (!oldCandidateNames.contains(candidateName)) {
               BeanDefinition bd = registry.getBeanDefinition(candidateName);
               if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                     !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                  candidates.add(new BeanDefinitionHolder(bd, candidateName));
               }
            }
         }
         candidateNames = newCandidateNames;
      }
   }
   while (!candidates.isEmpty()); // 用alreadyParsed来保证不会重复注册

   // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
   if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
      sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
   }

   if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
      // Clear cache in externally provided MetadataReaderFactory; this is a no-op
      // for a shared cache since it'll be cleared by the ApplicationContext.
      ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
   }
}
```

###  parse(解析)

首先看下parse方法，此处省略了部分代码，由于我们仅关注java config的方式，所以只看AnnotatedBeanDefinition分支即可

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
   for (BeanDefinitionHolder holder : configCandidates) {
      BeanDefinition bd = holder.getBeanDefinition();
      if (bd instanceof AnnotatedBeanDefinition) {
          parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
      }
   }
   this.deferredImportSelectorHandler.process();
}
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
		processConfigurationClass(new ConfigurationClass(metadata, beanName), DEFAULT_EXCLUSION_FILTER);
}
	protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
		if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}
		// 没走
		ConfigurationClass existingClass = this.configurationClasses.get(configClass);
		if (existingClass != null) {
			if (configClass.isImported()) {
				if (existingClass.isImported()) {
					existingClass.mergeImportedBy(configClass);
				}
				// Otherwise ignore new imported config class; existing non-imported class overrides it.
				return;
			}
			else {
				// Explicit bean definition found, probably replacing an import.
				// Let's remove the old one and go with the new one.
				this.configurationClasses.remove(configClass);
				this.knownSuperclasses.values().removeIf(configClass::equals);
			}
		}

		// Recursively process the configuration class and its superclass hierarchy.
		SourceClass sourceClass = asSourceClass(configClass, filter);
		do {
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
		}
		while (sourceClass != null);

		this.configurationClasses.put(configClass, configClass);
	}
```

### ConfigurationClass的beanMethods属性

经过层层调用，最终来到doProcessConfigurationClass方法，该方法是解析的核心步骤

```java
protected final SourceClass doProcessConfigurationClass(
      ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
      throws IOException {
   // @Configuration继承自@Component
   if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
      // 内部类处理，不用关注
      // Recursively process any member (nested) classes first
      processMemberClasses(configClass, sourceClass, filter);
   }
	
   // 省略以下代码 ... 
   // @PropertySource 注解处理
   // @ComponentScan 注解处理
   // @Import 注解处理
   // @ImportResource 注解处理

   // 核心逻辑，解析为BeanMethod，并添加到configClass的beanMethods属性中
   // Process individual @Bean methods
   Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
   for (MethodMetadata methodMetadata : beanMethods) {
      configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
   }
   // 如果配置类实现了接口，要执行接口里的default方法，可忽略
   // Process default methods on interfaces
   processInterfaces(configClass, sourceClass);

   // Process superclass, if any
   if (sourceClass.getMetadata().hasSuperClass()) {
      String superclass = sourceClass.getMetadata().getSuperClassName();
      if (superclass != null && !superclass.startsWith("java") &&
            !this.knownSuperclasses.containsKey(superclass)) {
         this.knownSuperclasses.put(superclass, configClass);
         // Superclass found, return its annotation metadata and recurse
         return sourceClass.getSuperClass();
      }
   }
   // No superclass -> processing is complete
   // 结束，停止外层循环
   return null;
}
```

retrieveBeanMethodMetadata是解析@Bean方法的底层逻辑，这里其实就是用反射获取被@Bean注解标注的方法，但是JVM反射获取的方法列表是无序的，而Spring内部实现是需要顺序的，这里用ASM技术来获取方法列表

```java
private Set<MethodMetadata> retrieveBeanMethodMetadata(SourceClass sourceClass) {
   AnnotationMetadata original = sourceClass.getMetadata();
   // 获取被@Bean标注的方法，JVM反射方式
   Set<MethodMetadata> beanMethods = original.getAnnotatedMethods(Bean.class.getName());
   // JVM返回的方法在顺序上是随机的，这里要用ASM来获取方法声明的顺序，关于ASM的两个类：ClassReader和ClassVisitor
   if (beanMethods.size() > 1 && original instanceof StandardAnnotationMetadata) {
      // Try reading the class file via ASM for deterministic declaration order...
      // Unfortunately, the JVM's standard reflection returns methods in arbitrary
      // order, even between different runs of the same application on the same JVM.
      try {
         AnnotationMetadata asm =
               this.metadataReaderFactory.getMetadataReader(original.getClassName()).getAnnotationMetadata();
         // 通过ASM解析出来的方法列表
         Set<MethodMetadata> asmMethods = asm.getAnnotatedMethods(Bean.class.getName());
         if (asmMethods.size() >= beanMethods.size()) {
            // 注意是LinkedHashSet
            Set<MethodMetadata> selectedMethods = new LinkedHashSet<>(asmMethods.size());
            // 两个for的作用：
            // 既是按asm方法列表的顺序添加，也保证了asm解析出来的方法一定在jvm解析的方法列表中，这样才能添加到返回结果中
            for (MethodMetadata asmMethod : asmMethods) {
               for (MethodMetadata beanMethod : beanMethods) {
                  if (beanMethod.getMethodName().equals(asmMethod.getMethodName())) {
                     selectedMethods.add(beanMethod);
                     break;
                  }
               }
            }
            if (selectedMethods.size() == beanMethods.size()) {
               // All reflection-detected methods found in ASM method set -> proceed
               beanMethods = selectedMethods;
            }
         }
      }
      catch (IOException ex) {
         logger.debug("Failed to read class file via ASM for determining @Bean method order", ex);
         // No worries, let's continue with the reflection metadata we started with...
      }
   }
   return beanMethods;
}
```

parse相关的处理到此结束，主要就是将@Bean方法解析成BeanMethod，并添加到ConfigurationClass的beanMethods属性中

### 验证与注册

首先还是说说为什么@Bean方法需要被增强，这是因为@Bean方法内部也可以调用别的@Bean方法，来实现bean的依赖，那么每调用一次就新增一个bean对象吗？这就会破坏bean的单例特性，因此就需要CGLIB增强来实现，这也是@Configuration的proxyBeanMethods默认为true的原因。当然如果bean没有被依赖，不被增强也是可以的，也就是Lite模式的Bean

验证的前提是proxyBeanMethods为true，步骤为：首先类不能是final的，不然无法增强，其次@Bean方法如果是静态的直接算通过，但如果不是静态的，而且是private final的，则会抛出异常

关于验证的源码如下：

```java
public void validate(ProblemReporter problemReporter) {
   // A configuration class may not be final (CGLIB limitation) unless it declares proxyBeanMethods=false
   Map<String, Object> attributes = this.metadata.getAnnotationAttributes(Configuration.class.getName());
   if (attributes != null && (Boolean) attributes.get("proxyBeanMethods")) {
      if (this.metadata.isFinal()) {
         problemReporter.error(new FinalConfigurationProblem());
      }
      for (BeanMethod beanMethod : this.beanMethods) {
         beanMethod.validate(problemReporter);
      }
   }
}
	public void validate(ProblemReporter problemReporter) {
		// @Bean 方法如果是静态，直接算验证通过
		if (getMetadata().isStatic()) {
			// static @Bean methods have no constraints to validate -> return immediately
			return;
		}

		// 如果是private，final的方法，无法被CGLIB重写
		if (this.configurationClass.getMetadata().isAnnotated(Configuration.class.getName())) {
			if (!getMetadata().isOverridable()) {
				// instance @Bean methods within @Configuration classes must be overridable to accommodate CGLIB
				problemReporter.error(new NonOverridableMethodError());
			}
		}
	}
```

#### 解析并注册BeanDefinition

验证通过之后，便是ConfigurationClassBeanDefinitionReader从ConfigurationClass的beanMethods属性中读取并解析BeanDefinition，相关源码在reader的loadBeanDefinitions--->loadBeanDefinitionsForConfigurationClass方法中

该方法我们主要看最后一个for循环中的loadBeanDefinitionsForBeanMethod方法

```java
private void loadBeanDefinitionsForConfigurationClass(
      ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

   // 用Condition来判断是否注册bean
   if (trackedConditionEvaluator.shouldSkip(configClass)) {
      String beanName = configClass.getBeanName();
      // 被跳过的就移除
      if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
         this.registry.removeBeanDefinition(beanName);
      }
      this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
      return;
   }

   if (configClass.isImported()) {
      registerBeanDefinitionForImportedConfigurationClass(configClass);
   }
   for (BeanMethod beanMethod : configClass.getBeanMethods()) {
      loadBeanDefinitionsForBeanMethod(beanMethod);
   }

   // 从@ImportResource中加载BeanDefinition
   loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
   // 调用ImportBeanDefinitionRegistrar的回调函数，其中是BeanDefinition的注册
   loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

loadBeanDefinitionsForBeanMethod完成了最终的解析与注册功能，主要是Bean的各项属性填充，然后进行注册

```java
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
		ConfigurationClass configClass = beanMethod.getConfigurationClass();
		MethodMetadata metadata = beanMethod.getMetadata();
		String methodName = metadata.getMethodName();

		// 关于Bean加载的条件判断，如Spring boot中的@ConditionalOnMissingBean
		// Do we need to mark the bean as skipped by its condition?
		if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
			configClass.skippedBeanMethods.add(methodName);
			return;
		}
		if (configClass.skippedBeanMethods.contains(methodName)) {
			return;
		}

		// 获取@Bean的元信息，包含里面各个属性值
		AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
		Assert.state(bean != null, "No @Bean annotation attributes");

		// Consider name and any aliases
		List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
		// 默认用方法名作为bean name
		String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

		// Register aliases even when overridden
		// 注册别名
		for (String alias : names) {
			this.registry.registerAlias(beanName, alias);
		}

		// Has this effectively been overridden before (e.g. via XML)?
		// 是否被重复注册了
		if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
			if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
				throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
						beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
						"' clashes with bean name for containing configuration class; please make those names unique!");
			}
			return;
		}

		ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
		beanDef.setResource(configClass.getResource());
		beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

		if (metadata.isStatic()) {
			// static @Bean method
			if (configClass.getMetadata() instanceof StandardAnnotationMetadata) {
				beanDef.setBeanClass(((StandardAnnotationMetadata) configClass.getMetadata()).getIntrospectedClass());
			}
			else {
				beanDef.setBeanClassName(configClass.getMetadata().getClassName());
			}
			beanDef.setUniqueFactoryMethodName(methodName);
		}
		else {
			// instance @Bean method
			beanDef.setFactoryBeanName(configClass.getBeanName());
			beanDef.setUniqueFactoryMethodName(methodName);
		}

		if (metadata instanceof StandardMethodMetadata) {
			beanDef.setResolvedFactoryMethod(((StandardMethodMetadata) metadata).getIntrospectedMethod());
		}

		beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
		beanDef.setAttribute(org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.
				SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

		AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

		Autowire autowire = bean.getEnum("autowire");
		if (autowire.isAutowire()) {
			beanDef.setAutowireMode(autowire.value());
		}

		boolean autowireCandidate = bean.getBoolean("autowireCandidate");
		if (!autowireCandidate) {
			beanDef.setAutowireCandidate(false);
		}

		String initMethodName = bean.getString("initMethod");
		if (StringUtils.hasText(initMethodName)) {
			beanDef.setInitMethodName(initMethodName);
		}

		String destroyMethodName = bean.getString("destroyMethod");
		beanDef.setDestroyMethodName(destroyMethodName);

		// Consider scoping
		ScopedProxyMode proxyMode = ScopedProxyMode.NO;
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
		if (attributes != null) {
			beanDef.setScope(attributes.getString("value"));
			proxyMode = attributes.getEnum("proxyMode");
			if (proxyMode == ScopedProxyMode.DEFAULT) {
				proxyMode = ScopedProxyMode.NO;
			}
		}

		// Replace the original bean definition with the target one, if necessary
		BeanDefinition beanDefToRegister = beanDef;
		if (proxyMode != ScopedProxyMode.NO) {
			BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
					new BeanDefinitionHolder(beanDef, beanName), this.registry,
					proxyMode == ScopedProxyMode.TARGET_CLASS);
			beanDefToRegister = new ConfigurationClassBeanDefinition(
					(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
		}

		if (logger.isTraceEnabled()) {
			logger.trace(String.format("Registering bean definition for @Bean method %s.%s()",
					configClass.getMetadata().getClassName(), beanName));
		}
    	// 注册BeanDefinition
		this.registry.registerBeanDefinition(beanName, beanDefToRegister);
	}
```

## postProcessBeanFactory

在看完postProcessBeanDefinitionRegistry的源码后，还有一个postProcessBeanFactory方法，这是BeanFactoryPostProcessor中的方法，它晚于postProcessBeanDefinitionRegistry执行。

该方法主要做了两件事：增强ConfigurationClass，添加了一个BeanPostProcessor——ImportAwareBeanPostProcessor，注意这是bean的后置处理器，和现在讲的bean factory后置处理器不是一个东西

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   int factoryId = System.identityHashCode(beanFactory);
   if (this.factoriesPostProcessed.contains(factoryId)) {
      throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + beanFactory);
   }
   this.factoriesPostProcessed.add(factoryId);
   if (!this.registriesPostProcessed.contains(factoryId)) {
      // BeanDefinitionRegistryPostProcessor hook apparently not supported...
      // Simply call processConfigurationClasses lazily at this point then.
      processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
   }
   	
   enhanceConfigurationClasses(beanFactory);
   beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
```

### 增强过程

```java
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
   Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
   for (String beanName : beanFactory.getBeanDefinitionNames()) {
      BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
      Object configClassAttr = beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE);
      MethodMetadata methodMetadata = null;
      if (beanDef instanceof AnnotatedBeanDefinition) {
         methodMetadata = ((AnnotatedBeanDefinition) beanDef).getFactoryMethodMetadata();
      }
      if ((configClassAttr != null || methodMetadata != null) && beanDef instanceof AbstractBeanDefinition) {
         // Configuration class (full or lite) or a configuration-derived @Bean method
         // -> resolve bean class at this point...
         AbstractBeanDefinition abd = (AbstractBeanDefinition) beanDef;
         if (!abd.hasBeanClass()) {
            try {
               abd.resolveBeanClass(this.beanClassLoader);
            }
            catch (Throwable ex) {
               throw new IllegalStateException(
                     "Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
            }
         }
      }
      if (ConfigurationClassUtils.CONFIGURATION_CLASS_FULL.equals(configClassAttr)) {
         if (!(beanDef instanceof AbstractBeanDefinition)) {
            throw new BeanDefinitionStoreException("Cannot enhance @Configuration bean definition '" +
                  beanName + "' since it is not stored in an AbstractBeanDefinition subclass");
         }
         else if (logger.isInfoEnabled() && beanFactory.containsSingleton(beanName)) {
            logger.info("Cannot enhance @Configuration bean definition '" + beanName +
                  "' since its singleton instance has been created too early. The typical cause " +
                  "is a non-static @Bean method with a BeanDefinitionRegistryPostProcessor " +
                  "return type: Consider declaring such methods as 'static'.");
         }
         configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
      }
   }
   if (configBeanDefs.isEmpty()) {
      // nothing to enhance -> return immediately
      return;
   }

   ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
   for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
      AbstractBeanDefinition beanDef = entry.getValue();
      // If a @Configuration class gets proxied, always proxy the target class
      beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
      // Set enhanced subclass of the user-specified bean class
      Class<?> configClass = beanDef.getBeanClass();
      Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
      if (configClass != enhancedClass) {
         if (logger.isTraceEnabled()) {
            logger.trace(String.format("Replacing bean definition '%s' existing class '%s' with " +
                  "enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
         }
         beanDef.setBeanClass(enhancedClass);
      }
   }
}
```

# 总结

本文主要分析了ConfigurationClassPostProcessor作为一个BeanDefinitionRegistryPostProcessor是如何将java config模式中的Bean解析BeanDefinition，并最终注册到容器中的，主要是以下思路：

因为ConfigurationClassPostProcessor实现的BeanDefinitionRegistryPostProcessor，因此我们要先看postProcessBeanDefinitionRegistry，再看postProcessBeanFactory，原因在前文也说了，invokeBeanFactoryPostProcessors是按此顺序执行的

之后Spring将注册BeanDefinition分为了3步：解析，验证，注册

1. 解析主要是将被@Configuration抽象成一个ConfigurationClass，将@Bean方法解析成一个BeanMethod，

然后添加到ConfigurationClass的beanMethods属性中

2. 验证是在使用CGLIB代理的前提下，验证类和方法是否能满足CGLIB增强的条件，比如final，private等
3. 注册是ConfigurationClassBeanDefinitionReader从ConfigurationClass的beanMethods属性中读取并解析成BeanDefinition，然后进行注册

因此我们可以总结出BeanDefinitionRegistryPostProcessor的主要作用就是从各种配置中解析BeanDefinition，并注册到容器中。



## 流程图

![bean_definition_registry](/Users/admin/mine/spring-pics/bean_definition_registry.png)
