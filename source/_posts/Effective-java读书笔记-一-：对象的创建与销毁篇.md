---
title: Effective java读书笔记(一)：对象的创建与销毁篇
date: 2018-11-23 10:31:35
categories: 读书笔记
tags: [Java基础]
toc: true
comments: true
---

# 对象的创建与销毁篇

## 使用静态工厂创建对象

创建一个对象最常用的方式是构造方法，但有时也要考虑使用静态工厂创建对象：

```java
public class UserFactory {
    
    public static User createUser() {
        User user = new User();
        user.setName("tom");
        return user;
    }
}
```



静态工厂的好处有三:
1. 有名称，不同于构造方法，静态工厂作为一个普通的静态方法，可以使用名称更清晰的表达作者的想法，对比多个构造方法的情况，往往读者看见多个不同参数类型，不同顺序的构造方法时，不知道它们是干什么的，静态工厂使用方法名提高了代码可读性，私以为在企业开发中，多人合作时，面对复杂的业务逻辑，可读性尤为重要
2. 不必每次都创建一个对象，或者说对象可以被重复利用，例如初始化一个数据库连接对象，不必每次设置用户名密码创建这个对象
3. 可以使用多态，返回子类类型的对象，例如通过参数来判断应该返回哪种子类型

同时它也是有缺点的，在看到构造方法时，我们能一眼看出是用于创建对象，但是静态工厂则不一定，因此静态工厂的方法名遵从一些惯用名称：valueOf，getInstance，newInstance等等



## 构造方法参数过多时，使用Builder

### Builder是什么

```java
User tom = User.builder().name("tom").age(18).build();
```



### 出现原因

静态工厂和构造方法都不能很好的解决参数过多时，参数是否必传问题，通常先写一个大而全的参数方法，然后提供多个部分参数方法。

以静态工厂为例，比如：有三个参数，其中address，age可不传，先写出一个三参数的，然后下面的方法传null来调用

```java
    public static User createUser(String name, Integer age, String address) {
        //... 
    }
    
    public static User createUser(String name, Integer age) {
        return createUser(name,age, null);
    }
    
    public static User createUser(String name, String address) {
        return createUser(name, null,address);
    }
```

这样做的弊端很明显，参数多了不利于扩展，不扩展又会导致调用者必须传一些无用的参数，并且代码难以阅读，调用方还容易出错



### 解决方案

#### 1.JavaBean

```java
User user = new User();
user.setXxx();
// ...
```

通过setter可以很好的避免上述问题，但书中所说JavaBean本身时可变类，无法成为不可变类，在这set的过程中有可能会产生线程安全问题，笔者认为实际业务开发中JavaBean多用于方法形参，属于线程私有，除非定义在成员变量位置，否则线程安全问题极低

#### 2.Builder模式

由于Builder模式代码编写很多，我们在实际开发中使用lombok可以更快的达到目的，事先引入依赖

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

JavaBean类上加入`@Builder`注解，即可像开头那样调用

```java
@Builder
public class User {
    private String name;
    private Integer age;
    private String address;
}
```



### 小结

如果类的构造方法或者静态工厂有多个参数时，优先考虑Builder模式，特别是某些参数可选时



## 使用枚举来创建单例模式

使用工厂模式创建单例模式分为：懒汉式，饿汉式。使用枚举作为替代主要有以下两个原因

1. 懒汉式通常需要与double-check配合使用来保证线程安全，而枚举本身就是线程安全的
2. 工厂模式在对象反序列化无法保证单例，需要重写readResolve，而枚举自动实现了反序列化

### 使用枚举创建User单例

```java
public enum UserSingleton {
    INSTANCE;

    private User user;

    UserSingleton() {
        this.user = new User();
    }

    public String showName(){
        return user.getName();
    }
}
```

### 枚举的线程安全性

枚举类在反编译之后，是一个不可变类，因此它是线程安全的


### 测试饿汉式的反序列化失效情况
使用饿汉式创建User单例模式类,并为`User`类实现`Serializable`接口

```java
public class UserSingletonFactory {
    private UserSingletonFactory(){}
    private static User user = new User();
    public static User getInstance() {
        return user;
    }
}
```
测试类

```java
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        User instance = UserSingletonFactory.getInstance();
        User other = UserSingletonFactory.getInstance();
        // 此时单例模式的结果返回true
        System.out.println(instance == other);
        
        ObjectOutputStream oos =
                new ObjectOutputStream(new FileOutputStream(new File("obj.txt")));
        oos.writeObject(instance);
        oos.close();

        ObjectInputStream ois =
                new ObjectInputStream(new FileInputStream(new File("obj.txt")));
        User user = (User) ois.readObject();
        ois.close();

        System.out.println(instance == user);
    }
```

结果最后的输出为false



## 通过私有的构造方法让类不可实例化

### 解释

1. 私有的构造方法，指用private修饰构造方法
2. 不可实例化，通过私有的构造方法，让类无法产生对象

### 使用场景

作为一些工具类，例如JDK中的`Math`类，只希望使用它的静态成员变量和静态方法，所以我们可以看到源码中的Math类构造方法如下

```java
public final class Math {

    /**
     * Don't let anyone instantiate this class.
     */
    private Math() {}
}
```



## 避免创建不必要的对象

原则：尽量重用对象，如果是不可变类产生的对象，那它始终可以被重用

典型的不可变类如`String`,只要是相同的字符串，内存中只有一个String对象

同时有静态工厂和构造器的不可变类，优先使用静态工厂创建对象，静态工厂不会重复创建对象，如

```java
Boolean.valueOf("true");
```

优先于

```java
new Boolean("true");
```

除了不可变对象，还可以重用那些被认为不会被修改的可变对象，书中用一个日期类举例，将只需要初始化一次的对象放在静态代码块中，在实际开发中，诸如数据库连接池，http client请求线程池等重量级的对象，为了提高性能，必须重用


## 清理过期的对象引用

为了防止内存泄漏，需要将不再使用的对象，解除引用，即obj = null，将引用指向空，让GC回收对象

例如List的remove方法中的代码片段

```java
elementData[--size] = null; // clear to let GC do its work
```


需要注意的是清除对象引用是在特殊情况下的处理，并不是一种规范，我们在实际开发中并不需要小心翼翼的处理

### 何时清除对象引用

如果类自己管理内存空间，如`ArrayList`内部使用`Object`数组存储，一旦其中的元素被删除，则需要清空对象引用



### 避免使用finalize方法

老生常谈的`finalize`方法问题，不要尝试调用它，GC并不会立即回收对象，甚至不保证执行。经过测试，调用finalize还会降低性能，花费更多的时间销毁对象，书中后面讲解的内容实用性太低，不做记录