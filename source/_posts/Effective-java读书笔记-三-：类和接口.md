---
title: Effective-java读书笔记(三)：类和接口
date: 2018-11-28 22:11:35
categories: 读书笔记
tags: [Java基础]
toc: true
comments: true
---

# 使类和接口的可访问性最小化

模块设计原则：对外隐藏内部数据和实现细节，把api和他的实现隔离开来，模块之间通过api通信，一个模块不需要知道其它模块的内部细节，这称之为封装。

封装有效地让各模块直接解耦，解耦之后模块可以独立的开发，测试，优化，使用及修改。


1. 尽可能地使每个类或者成员不被外界访问
   1. 类或接口尽可能的做成包级私有的，在以后的版本中，可以对他修改
   2. 如果你把类做成公有的，你就有责任永远对它负责，保证后续版本的兼任性
   3. 如果一个类只在某个类中使用，则考虑使用嵌套类(nested-class)

如果

















