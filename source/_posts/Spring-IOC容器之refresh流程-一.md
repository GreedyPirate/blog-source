---
title: Spring IOC容器之refresh流程(一)
date: 2020-02-05 18:49:56
categories: Spring
tags: [Spring,源码]
toc: true
comments: true
---

# 前言

上文讲到了AnnotationConfigApplicationContext初始化的前两步，本文开始重点讲解第三步refresh。

# 类图

AbstractApplicationContext是所有ApplicationContext实现类的抽象接口，在它的refresh方法中抽象出了绝大部分的代码，而子类只需要根据需要实现其中几个方法，用于不同子类的差异，或者为子类提供钩子函数。

![refresh_uml](/Users/admin/mine/spring-pics/refresh_uml.png)

注意，上文提到过GenericApplicationContext会创建DefaultListableBeanFactory

## 源码

refresh方法源码很多，我会拆分并省略部分不重要的代码。但有一点可以事先说明的是，refresh最重要的2个方法调用是invokeBeanFactoryPostProcessors和finishBeanFactoryInitialization

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
        // 暂时省略 ...
      }
      finally {
          // 暂时省略 ...
      }
   }
}
```

首先就是一个对象锁，防止多线程情况下重复执行refresh，第一个方法是prepareRefresh

## prepareRefresh

```java
protected void prepareRefresh() {
   // Switch to active.
   // 记录启动事件，并重置2个标志位表示容器启动 
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   // Initialize any placeholder property sources in the context environment.
   // AnnotationConfigApplicationContext这条分支没有实现，不关注
   initPropertySources();

   // Validate that all properties marked as required are resolvable:
   // see ConfigurablePropertyResolver#setRequiredProperties
   // 校验配置，目前需要校验的参数为0，暂不关注
   getEnvironment().validateRequiredProperties();

   // 创建需要在refresh之前就需要监听的事件及监听器集合
   // Store pre-refresh ApplicationListeners...
   if (this.earlyApplicationListeners == null) {
      this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
   }
   else {
      // Reset local application listeners to pre-refresh state.
      this.applicationListeners.clear();
      this.applicationListeners.addAll(this.earlyApplicationListeners);
   }

   // Allow for the collection of early ApplicationEvents,
   // to be published once the multicaster is available...
   this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

### obtainFreshBeanFactory

该方法主要从子类中获取beanFactory，并刷新beanFactory。刷新由GenericApplicationContext实现，主要是设置刷新标志位，并设置序列化Id，此处的id为当前AnnotationConfigApplicationContext对象的内存地址值

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}

// GenericApplicationContext实现
protected final void refreshBeanFactory() throws IllegalStateException {
   if (!this.refreshed.compareAndSet(false, true)) {
      throw new IllegalStateException(
            "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
   }
   this.beanFactory.setSerializationId(getId());
}
```

### prepareBeanFactory

该方法的代码很多，通过注释看，主要还是准备工作，我们需要重点关注的是注册的BeanPostProcessor，这里有3个：ApplicationContextAwareProcessor，ApplicationListenerDetector，LoadTimeWeaverAwareProcessor。分别对应我们平时用到的Aware接口，事件监听以及AOP

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		// 设置用于加载bean的ClassLoader，默认是当前线程的context ClassLoader
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		// 注册了一个BeanPostProcessor，主要是为bean提供EnvironmentAware，ApplicationContextAware等接口的实现
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		// 忽略依赖注入的bean，通常是容器内部用的
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// 实现spring事件监听机制用的，ApplicationListener的BeanPostProcessor
		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		// AOP相关，LTW织入的BeanPostProcessor
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// 注册环境相关的单例对象
		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

prepareBeanFactory的分析到此为止，接下来的postProcessBeanFactory方法在子类中没有重写，不用关注，直接看invokeBeanFactoryPostProcessors。



### invokeBeanFactoryPostProcessors

该方法很简单，首先是调用所有的BeanFactoryPostProcessor，后面的if可以不用关注，因为在prepareBeanFactory方法已添加过了，这里不会进if

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

   // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
   // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
   // Aspectj LWT 类加载时织入
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

#### PostProcessorRegistrationDelegate调用BeanFactoryPostProcessor

在看该方法之前，先看看以下2个类图。

DefaultListableBeanFactory是默认的BeanFactory实现，但同时它也实现了BeanDefinitionRegistry接口；那么BeanFactoryPostProcessor对应的就有一个BeanDefinitionRegistryPostProcessor

![factory_processor](/Users/admin/mine/spring-pics/factory_processor.png)

#### 源码分析

invokeBeanFactoryPostProcessors方法的源码如下，首先我们看一下注释中的第一部分。

第一部分主要是执行BeanFactoryPostProcessor，前面我也说了，有一种特殊的BeanFactoryPostProcessor——BeanDefinitionRegistryPostProcessor，通过名字我们也知道它是具有BeanDefinition注册功能的后置处理器，

这里需要我们细心观察下，是执行了BeanDefinitionRegistryPostProcessor，但是普通的BeanFactoryPostProcessor只是添加进了regularPostProcessors

还有十分重要的一点是，绝大部分的PostProcessor都不会在这里执行，因为必须是通过AbstractApplicationContext#addBeanFactoryPostProcessor添加进来的BeanFactoryPostProcessor才会在第一分部执行，比如我们常用的Spring boot通过实现ApplicationContextInitializer接口，就是调用了addBeanFactoryPostProcessor方法添加进来的PostProcessor就会执行

##### 执行顺序

上述的BeanFactoryPostProcessor集合是方法参数里传进来的，何时添加的上面也说过了，下面的BeanFactoryPostProcessor集合则是从容器里获取的，这类BeanFactoryPostProcessor又按执行顺序分为三类：

1. 实现了PriorityOrdered的BeanDefinitionRegistryPostProcessor
2. 实现了Ordered的BeanDefinitionRegistryPostProcessor
3. 其它BeanFactoryPostProcessor

第二部分的源码主要就是从容器中获取并按顺序执行了这三类BeanFactoryPostProcessor，也可以看出先执行的是BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法，最后才执行BeanFactoryPostProcessor的postProcessBeanFactory方法

注意类图中BeanDefinitionRegistryPostProcessor继承自BeanFactoryPostProcessor的关系

```java
public static void invokeBeanFactoryPostProcessors(
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

   // Invoke BeanDefinitionRegistryPostProcessors first, if any.
   Set<String> processedBeans = new HashSet<>();
	
   // ===================== 第一部分 ============================
   // DefaultListableBeanFactory继承了BeanDefinitionRegistry
   if (beanFactory instanceof BeanDefinitionRegistry) {
      BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
      List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
      List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

      // 这里调用的是通过AbstractApplicationContext#addBeanFactoryPostProcessor添加进来的BeanDefinitionRegistryPostProcessor
      // 例如Spring boot通过实现ApplicationContextInitializer接口添加了很多BeanDefinitionRegistryPostProcessor
      for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
         if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            BeanDefinitionRegistryPostProcessor registryProcessor =
                  (BeanDefinitionRegistryPostProcessor) postProcessor;
            // 执行
            registryProcessor.postProcessBeanDefinitionRegistry(registry);
            registryProcessors.add(registryProcessor);
         }
         else {
            // 这里是普通的BeanFactoryPostProcessor，仅仅添加到了集合，并没有开始执行
            regularPostProcessors.add(postProcessor);
         }
      }

       // ===================== 第二部分 ============================
      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the bean factory post-processors apply to them!
      // Separate between BeanDefinitionRegistryPostProcessors that implement
      // PriorityOrdered, Ordered, and the rest.
      List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

      // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.

      // 先调用的是实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessor
      String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);

      // 这里就包含了ConfigurationClassPostProcessor，它基本解决了绝大部分bean的BeanDefinition解析以及注册
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      // 该阶段结束，清空一次
      currentRegistryProcessors.clear();

      // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
      // 第二批处理的是实现了Ordered的BeanDefinitionRegistryPostProcessor
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
      boolean reiterate = true;
      while (reiterate) {
         reiterate = false;
         postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
         for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName)) {
               currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
               processedBeans.add(ppName);
               reiterate = true;
            }
         }
         sortPostProcessors(currentRegistryProcessors, beanFactory);
         registryProcessors.addAll(currentRegistryProcessors);
         invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
         currentRegistryProcessors.clear();
      }

      // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
      // 为什么还要再执行一次BeanDefinitionRegistryPostProcessor，里面有判断，执行过的就不再执行了，所以不会重复执行
      invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);

      // 执行普通的BeanFactoryPostProcessor
      invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
   }

   else {
      // Invoke factory processors registered with the context instance.
      // 没有实现BeanDefinitionRegistry的BeanFactory，在这里调用BeanFactoryPostProcessor
      invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let the bean factory post-processors apply to them!
   String[] postProcessorNames =
         beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

   // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      // 这里会跳过上面已经执行的BeanFactoryPostProcessor
      if (processedBeans.contains(ppName)) {
         // skip - already processed in first phase above
      }
      else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   // Finally, invoke all other BeanFactoryPostProcessors.
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
   for (String postProcessorName : nonOrderedPostProcessorNames) {
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

   // Clear cached merged bean definitions since the post-processors might have
   // modified the original metadata, e.g. replacing placeholders in values...
   beanFactory.clearMetadataCache();
}
```

#### 小结

invokeBeanFactoryPostProcessors方法看似代码很多，但一言以蔽之，就是执行了所有的BeanFactoryPostProcessor，在当前的测试方法中，BeanFactoryPostProcessor的实现类只有一个：ConfigurationClassPostProcessor，下文主要讲解该类