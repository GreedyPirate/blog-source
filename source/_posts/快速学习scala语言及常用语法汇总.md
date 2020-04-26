---
title: 快速学习scala语言及常用语法汇总
date: 2019-11-27 23:03:09
categories: Kafka Tutorial
tags: [scala,kafka]
toc: true
comments: true
---

>阅读kafka源码的一大障碍就是scala语言，本文的目的是罗列kakfa源码中涉及到的基础源码知识，一些不常用的东西不会涉及，同时我也会不断总结遇到的语法特性，因为极个别语法俺也不会！

# 环境
和java一样，scala也需要sdk来运行程序，scala也是jvm语言，请事先保证本机上已安装java

## sdk
scala sdk的安装我强烈推荐sdkman(mac用户)，windows用户自己百度安装即可
sdk安装scala命令： sdk i scala 2.12.10

## ide
ide依然选择idea，安装scala插件即可

# 语法基础

## 启动类定义

main方法只能定义在object中，object中所有的方法和变量都是静态的

## 小知识点

一些小知识点提前说明
1. scala中句尾不用写分号
2. scala中的Unit就是java中的void

## 变量定义

var是变量，val是常量，常量利于jvm的GC回收，因此kafka中几乎都是val

同时scala中变量是不要定义类型的，会进行自动推断
这种语法比较恶心的是方法调用后的返回值类型不能一眼看出，比如
```java
	val obj = fun()
```
obj的类型通常要进入fun方法定义，看它的返回值

## if/else对变量赋值
```
val result = if(size > 0) true else false
```
看到这里是不是感觉神清气爽，想想java里要如何实现这个功能，真是省了不少事

## 打印字符串
scala中打印时拼接字符串极为方便
```java
log.info(s"request size is: ${request.size}")
```
s开头，后面跟字符串，打印变量，用${}包裹即可，java中几乎没有比这种方法更好的，即使String.format也很丑陋，更不要说加号拼接了


## 方法

首先记住一点，看见def就是方法定义，它的大概形式如下
```java
def fun(参数名 :参数类型) :返回值类型 = {
	// 方法体...
}
```
scala中的方法千变万化，但是都是以上形式，鉴于篇幅，本文只介绍kafka中常用的一些形式

### 返回值

scala中方法返回值不用写return，最后一行代码就是返回值，遇到if/else,case也不要慌，慢慢看即可

#### 多个返回值

java中的方法不能有多个返回值，一直是我很耿耿于怀的地方
假如要返回方法中的一个List对象和一个Map对象，只能定义一个对象来保存

而scala的多返回值完全没有这种麻烦，还可以自行指定要哪个返回值
```java
def fun():(Int, Int) = {
	(10,20)
}

// 只要fun方法的第一个返回值
var result = fun()._1
```
以上用法在kafka源码中也会见到


### 省略

scala中的方法有各种省略。只有一行代码可以省略{}，返回值类型用自动推断也可以省略。如下：
```java
def getSize = size
```

### 嵌套方法

嵌套方法的意思是说方法里可以定义方法，常见的格式如下
```java
def handle() :Unit = {
	def callback() :Unit={
		//...
	}
	read(args..., callback)
}
```
以上例子是kafka中常见的一种用法：将回调方法先定义好，然后作为参数传给另一个方法
这种特性的好处也是显而易见的，配合函数传参使方法调用以及定义更优雅，想想java中要实现类似的功能，得要定义一个接口
坏处在于滥用该特性，嵌套多层或者定义太多嵌套方法，使代码阅读性变差

### 柯里化函数

听上去高大上的一个名字，其实既简单又实用，kafka中常见的定义如下
```java
def inLock(lock :Lock)(fun :Any){
	lock.lock()
    try {
      fun
    } finally {
      lock.unlock()
    }
}
```
这是kafka中的一个加锁方法的大概源码，它是一个公共方法，作用是传入一个锁和一个方法对象，然后在方法执行前后加锁和释放锁，这有些类似java中aop的思想

#### 如何调用
以上的柯里化函数调用方式在kafka源码中也采取了一定的简化
```java
inLock(readLock) {
	// ...
}
```
可以看到第二个参数的方法对象，直接用花括号


## 集合
scala中的集合默认是不可变的，所以经常看到mutabl开头的初始化，例如

```java
val map = mutable.HashMap[String, Integer]
```

## Option与match...case
kafka源码中充斥着大量的这种结果，Option和java中的Optional类似，都是为了避免空指针而引入的
在scala中Option有2个子类：Some和None，就是元素是否为空的2种类型

以下是从kafka源码中复制的一段代码
```java
partition.getReplica(replicaId) match {
	case Some(replica) =>
	  partition.updateReplicaLogReadResult(replica, readResult)
	case None =>
	  updatedReadResult = readResult.withEmptyFetchInfo
}
```
解释如下：
1. partition.getReplica(replicaId)的返回值类型是Option[Replica]
2. match...case类似于java中的switch...case
3. Some(replica)中的replica变量名是可以随便起的
总的来说类似 if(replica != null) ... else ...


# 特殊语法汇总

scala中求多个集合的并集，有3个List集合，list1，list2，list3
val all = list1 ++ list2 ++ list3





















