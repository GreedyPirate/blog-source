---
title: Effective java读书笔记(二)：对象通用方法
date: 2018-11-26 10:48:02
categories: 读书笔记
tags: [Java基础]
toc: true
comments: true
---

# 对象通用方法

## 覆盖equals时的约定

当类具有特有的"逻辑相等"概念时，必须覆盖equals方法，这样也可以使这个类作为map的key，或者set中的元素

当对象非null时，equals方法满足以下四个特性：

1. 自反性：`x.equals(x)=true`
2. 对称性：`x.equals(y)=true`时，`y.equals(x)`必须为true
3. 传递性：x=y，y=z，则x=z
4. 一致性：`x.equals(y)`在多次调用后返回相同的值

理解即可，不必要记忆

###         高效的编写equals方法

首先了解一个小知识点

```java
null instanceof Object = false
```

null不属于任何一个类型，所以对equals方法传入的对象不必做空指针判断

```java
    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof MyClass) {
            MyClass castObj = (MyClass)anObject;
        	// 自己的判断逻辑
            return true;
        }
        return false;
    }
```

#### 编写步骤

1. 首先用==检查参数是否是当前对象
2. 判断参数是否是要判断的类型，
3. 将对象强转成要比较的对象类型
4. 根据类中的字段编写自己的判断逻辑，返回相应的true或false



### ==与equals

相信很多新手同学分不清这二者的区别，以及使用场景，二者区别如下：

1. 先说基本类型的==判断：值相等就返回true

2. 再说引用类型的==：指向同一个对象才返回true

3. 最后是equals：在Object里，它和==时一样的，但是类可以有自己的判断依据，比如String类

#### 包装类的比较

问题：`Integer i = 1, Integer j = 1`,如何比较二者是否相等?

答案是`i.equals(j)`,切不可写成`i==j,因为Integer内部采用了缓存，-128至127之间的数字被视为同一个对象，此时是可以通过==判断两个数字是否相等，但这只是假象，超过这个区间的数字就会返回`false





## 覆盖equals是同时覆盖hashcode

### 约定

Java规范中包含以下约定：

1. 只要equals中用来判断两个对象是否相等的字段没有发生改变，那么调用多少次返回的结果都应该相同
2. 如果通过equals判断出两个对象相等，那么它们的hashcode方法的返回值一定相等；如果不相等，那么hashcode方法的返回值不一定不等，但这必然降低了散列表的性能

### 编写hashcode方法

hashcode方法编写的好坏，直接影响对象能否在集合中均匀分布，具体的编写方法见书41页，这里记下注意的几点：

1. 冗余字段不参与计算与比较，例如单价，数量，总价三者的关系，很明显总价可以通过另外二者计算出来，那么总价不必参与计算hashcode的过程，同时必须也不能参数equals的比较过程



## 覆盖toString方法

toString方法的作用显而易见，如果不覆盖Object中的toString方法，返回`类名@对象hashcode十六进制值`的表现方式

```java
getClass().getName() + "@" + Integer.toHexString(hashCode())
```

在实际开发中，toString用于记录日志是必不可少，例如打印用户信息，如果输出原始形式，则毫无价值，我们更关系的是用户id，用户名等关键信息



## 谨慎覆盖clone方法



## 考虑实现Comparable接口





























