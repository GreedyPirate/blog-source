---
title: SpringMVC源码分析
date: 2019-10-05 10:22:44
categories: Spring
tags: [Spring,Spring 扩展]
toc: true
comments: true
---

>曾经debug了一次SpringMVC的源码，但是平时比较忙(lan)，一直没有放在博客上，现在忙里偷闲整理上来

# DispatcherServlet#doDispatch方法分析

查找HandlerExecutionChain，它的名称为mappedHandler

![](https://ae01.alicdn.com/kf/H16c856f002a54ed2978dc4b1d33f2a6bD.png)

遍历HandlerMapping集合handlerMappings, 根据HandlerMapping的getHandler方法查找HandlerExecutionChain

![](https://ae01.alicdn.com/kf/H67d3c69356214afa9c0e7c328d1badedW.png)

HandlerMapping 接口的实现类AbstractHandlerMapping提供getHandler方法

![](https://ae01.alicdn.com/kf/H71b8c8a53ce94783a7c9b8a34676915e3.png)

HandlerMapping，AbstractHandlerMapping和AbstractHandlerMethodMapping三者之间的关系如下

![](https://ae01.alicdn.com/kf/Hc152e4b7c20e433b8cc4a74a3935115ef.png)

其主要实现流程如下：
1> 依靠getHandlerInternal方法获取对应的handler，handler指的就是Controller类中的方法，它处理接口请求
该方法由AbstractHandlerMapping 的子类AbstractHandlerMethodMapping提供

![](https://ae01.alicdn.com/kf/H3025ea329acb4328a6415047a3da6a6fD.png)

2> 由lookupHandlerMethod方法查找HandlerMethod

![](https://ae01.alicdn.com/kf/Hbe25bad47d0845d98b9f659464346fa8N.png)

找到合适的HandlerMethod并组装成一个Match对象，包装了url，请求方式和HandlerMethod对象

![](https://ae01.alicdn.com/kf/H7fc9dfd40f724de6b0ed432a0d9e017fA.png)

MappingRegistry中的*mappingLookup*是所有url和HandlerMethod的映射关系，最终组成了Match对象，如果发现有多个就取第一个作为bestMatch，并返回HandlerMethod

![](https://ae01.alicdn.com/kf/Ha2630f9cebab45678ef9e84e6247eff7e.png)

3> getHandlerInternal方法的最后，createWithResolvedBean方法是在初始化HandlerMethod中的Handler，它定义为Object bean，如果是String类型，就从BeanFactory中获取，最后（new HandlerMethod(this,handler)

getHandler方法执行完getHandlerInternal获取到HandlerMethod之后，获取HandlerExecutionChain
`HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request)`，

getHandlerExecutionChain在HandlerExecutionChain里填充了所有的HandlerInterceptor

# 继续doDispatch
找到HandlerExecutionChain后，然后查找HandlerAdapter，实现抽象类为AbstractHandlerMethodAdapter
![](https://ae01.alicdn.com/kf/Hb81c2ffa03154ba09a88839a6e2e357eh.png)
接下来判断请求方式为GET或HEAD，满足*Last-Modified*的请求直接返回
前文提到过HandlerExecutionChain里包含了所有的HandlerInterceptor，HandlerExecutionChain的applyPreHandle方法用于执行每个HandlerInterceptor的preHandle方法
如果有一个没通过，返回了false，同时出发afterCompletion，那么整个请求结束
最后HandlerAdapter的实现类RequestMappingHandlerAdapter调用handle开始处理请求，注释里也说明了”真正开始调用handler“

HandlerAdapter类图如下
![](https://ae01.alicdn.com/kf/Hac81cd901f654da69db0976c91765d6fw.png)
HandlerAdapter ha调用handle的流程如下，核心处理过程在其子类RequestMappingHandlerAdapter的handleInternal方法中调用的invokeHandlerMethod中(红框处)
![](https://ae01.alicdn.com/kf/H14d4d6d09b8d4716ad79def79ddd48d3D.png)
![](https://ae01.alicdn.com/kf/Hcfc0a4cb73f24d6ea49d169831f5c2dbx.png)
![](https://ae01.alicdn.com/kf/Ha5a165c031ff4de897937582f4e25dbd8.png)

## invokeHandlerMethod方法解析
![](https://ae01.alicdn.com/kf/H870bfbf42e454aa9aafb69c8d7fbe459A.png)

invokeHandlerMethod的方法很长，但重点关注红框内的代码,首先HandlerMethod类图的某一条类分支如下

![](https://ae01.alicdn.com/kf/H4b768a2b8d0b4806b3c6c9a277759ae2A.png)

InvocableHandlerMethod翻译为可调用的HandlerMethod，里面有三个字段
1. WebDataBinderFactory： 用于创建 WebDataBinder的工厂，而WebDataBinder主要用于请求数据和方法参数的绑定
2. HandlerMethodArgumentResolverComposite：一组HandlerMethodArgumentResolver的集合，HandlerMethodArgumentResolver主要用于参数解析
3. ParameterNameDiscoverer：用于获取方法或者构造方法的参数名称(很有意思的东西，普通反射等手段是获取不到的)

ServletInvocableHandlerMethod:多了个HandlerMethodReturnValueHandlerComposite，里面是一组HandlerMethodReturnValueHandler，而HandlerMethodReturnValueHandler用于处理返回值

第二处红框内的ServletInvocableHandlerMethod invocableMethod.invokeAndHandle(webRequest, mavContainer) 方法为入口
第一行代码就已经处理结束，获取了返回值，可以预料它的处理过程很长，注意上层没有providedArgs参数
![](https://ae01.alicdn.com/kf/H8230848f16fb403f907332bb61261ea8U.png)

而真正的调用过程都在ServletInvocableHandlerMethod的父类InvocableHandlerMethod中，过程只有两行代码，解析参数，调用方法，最后返回结果
![](https://ae01.alicdn.com/kf/Hcc1af06c535c469b90a0cd21e322dac1I.png)

## 参数解析
承接上文，getMethodParameters请request中获取参数，那么具体过程是如何的呢
![](https://ae01.alicdn.com/kf/H0cc6db5534574e20a2a71375af5e4cedM.png)
1. 首先MethodParameter是spring对方法参数的抽象封装， 可以理解为Method或者Constructor(二者共同父类为Executable) + Parameter，以及参数index，所在的class，参数类型，参数的泛型类型，参数的注解，参数名称
2. 遍历参数数组，由于providedArgs为空，所以暂时忽略args[i] = resolveProvidedArgument(parameter, providedArgs);
3. 判断是否可以解析这个参数，HandlerMethodArgumentResolver可以是多个，但是每个参数都有对应的Resolver解析，具体就由supportsParameter方法判断，前文提到HandlerMethodArgumentResolverComposite(即argumentResolvers)是一个HandlerMethodArgumentResolver集合，Composite的supportsParameter就是遍历里面的HandlerMethodArgumentResolver，通过Resolver的supportsParameter方法找到合适Resolver，如果找不到就说明不支持
![](https://ae01.alicdn.com/kf/H30c46b35c7244e81b925361a300f5608F.png)
![](https://ae01.alicdn.com/kf/H3e58404a2dd7408bb633d5123cbd96aaR.png)
4. resolveArgument也是在HandlerMethodArgumentResolverComposite中执行的
	`args[i] = this.argumentResolvers.resolveArgument(parameter, mavContainer, request, *this*.*dataBinderFactory*);`
视为过渡代码
![](https://ae01.alicdn.com/kf/Hfb9706db3fb74f708636a8a40ad2fe4cN.png)

接下来就是HandlerMethodArgumentResolver的解析了，其关键实现类为RequestResponseBodyMethodProcessor，它同时继承了ReturnValueHandler和ArgumentResolver接口，之后的很多地方都用到了它和它的Abstractxxx父类
![](https://ae01.alicdn.com/kf/H9045100b3ee1498cadf9ae26a09db0dem.png)

RequestResponseBodyMethodProcessor解析过程如下，首先就是MessageConverters解析，这里spring用了三个readWithMessageConverters，看得我眼晕，吐槽
![](https://ae01.alicdn.com/kf/H18b244aea3614aa3993484b20e0e9f83m.png)

第二个红框出的readWithMessageConverters是在RequestResponseBodyMethodProcessor的父类AbstractMessageConverterMethodArgumentResolver中
该方法有点长，只截取关键代码部分，简单来说就是遍历所有的messageConverter，根据messageConverter 的canRead方法判断是否要解析参数，然后根据getAdvice方法返回的RequestResponseBodyAdviceChain分别在解析前后执行
RequestResponseBodyAdviceChain比较简单，里面是一组RequestBodyAdvice集合和一组ResponseBodyAdvice集合，它们允许用户对请求参数和响应结果做修改(希望统一封装项目请求参数和返回值的童鞋看黑板)
最后返回的body就是方法参数对象
![](https://ae01.alicdn.com/kf/H9381d3a466414d969964de3795b8bb44A.png)

获取到参数后，回到RequestResponseBodyMethodProcessor的resolveArgument方法，这里的arg就是刚才返回的body，然后我们要把这个Object绑定到Controller方法里的参数对象上去，绑定之前要通过JSR303校验，因此validateIfApplicable主要就是做校验的，如果校验不通过，BindingResult就会携带着异常抛出，请求结束
![](https://ae01.alicdn.com/kf/H81e2c8aa48924025bd24721318885302v.png)

校验过程
![](https://ae01.alicdn.com/kf/Hca013c6334704efcbbefa03bb25a93beY.png)

### 调用
当解析，校验，绑定参数完成之后，便可以开始调用方法了，InvocableHandlerMethod#invokeForRequest继续执行
![](https://ae01.alicdn.com/kf/H5b71d02a027e4dd4923fc58f5d32cde17.png)
调用过程十分简单
1. 如果方法不是public，就先setAccessible(*true*)
2. 通过反射执行方法，getBean() 返回的是Controller对象
![](https://ae01.alicdn.com/kf/H3792b5c8d7a14038bdda3a619411f329m.png)

ServletInvocableHandlerMethod#invokeAndHandle在调用完父类InvocableHandlerMethod的invokeForRequest方法后，开始处理返回结果
![](https://ae01.alicdn.com/kf/He14f1031b43c4275b7f0502e60853b792.png)

返回结果由HandlerMethodReturnValueHandlerComposite 中某一个的HandlerMethodReturnValueHandler处理，根据supportsReturnType决定谁处理
然后发现还是交给了RequestResponseBodyMethodProcessor处理，因为它不仅是HandlerMethodArgumentResolver
还是HandlerMethodReturnValueHandler的实现类，也就是说解析参数，处理返回值的活都是它干的
![](https://ae01.alicdn.com/kf/Hc6518f793c654369a1a417e3f0b3a71dB.png)
![](https://ae01.alicdn.com/kf/H3da3d4369222482fa6674e59eefc05beb.png)

RequestResponseBodyAdviceChain中的ResponseBodyAdvice集合，根据各自的supports方法，判断要不要处理这个返回结果
然后再有真正的messageConverter处理，这里由AbstractGenericHttpMessageConverter#write
交给AbstractJackson2HttpMessageConverter#writeInternal做json序列化处理
![](https://ae01.alicdn.com/kf/H3ec8bbe3dbde4c779d60354c83735b9dP.png)

# 结束

剩下的代码大部分都索然无味了，一路跳回到DispatcherServlet的doDispatch方法
![](https://ae01.alicdn.com/kf/Had821ec5f1144629949b04a78dd6b6fdc.png)
这里执行了所有HandlerInterceptor的postHandle方法


