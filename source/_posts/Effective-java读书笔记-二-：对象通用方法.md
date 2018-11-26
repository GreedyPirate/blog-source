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
4. 编写自己的判断逻辑，返回相应的true或false


