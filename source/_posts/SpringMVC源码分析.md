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
![](https://ae01.alicdn.com/kf/Hbe25bad47d0845d98b9f659464346fa8N.png)
![](https://ae01.alicdn.com/kf/H67d3c69356214afa9c0e7c328d1badedW.png)
![](https://ae01.alicdn.com/kf/H16c856f002a54ed2978dc4b1d33f2a6bD.png)
![](https://ae01.alicdn.com/kf/H3025ea329acb4328a6415047a3da6a6fD.png)
