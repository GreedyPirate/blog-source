---
title: java 8日期类学习
date: 2019-01-18 11:31:10
categories: 技术积累
tags:
	- [积累]
toc: true
comments: true
---

>最近开发过程中遇到了很多时间类处理，由于对Calender类不熟悉，我说这个类设计的烂，谁赞成，谁反对？也被推荐过joda-time类库，鉴于项目用的都是java 8了，是时候了解一下java.time包下的类了

# 导言

## java 8 日期类的优势
用完java 8的API之后，只有一个感觉，爽，没有啰嗦的方法，很多静态工厂方法，of，from见名知意，用到后面，一些api自己都可以猜出来了，同时api更人性化，例如对比之前的获取月份方法，是从0开始，机械化的思维，反观java 8，我能通过getMonthValue直接获取，无须+1

对于joda-time，从个人角度讲，实在不想再去记忆一套api，同时从项目角度来说，我能用jdk实现的，为什么要依赖第三方jar包，这点对于实际开发来说更重要。但对于还在使用java 8以下版本的同学joda-time还是值得推荐的

## API的记忆方法
在Effective java读书笔记一文中，静态方法相较构造方法有更多的优势，尤其是在提供给开发者使用时。java 8这方面做得很好，of，from，parse，format，minus，plus等等方法，一眼就知道作者的意图

## 时间分类
java 8提供了3个基础时间类：LocalDate, LocalDateTime, LocalTime，分别代表日期，日期+时间，时间(时分秒)

同时三者之间可以部分转换，之所以称之为部分，很简单的例子是日期无法直接转换为具体的日期+时间，因为它缺少时分秒，这可以理解为一种精度损失，当然你可以通过默认值来补全

Instant表示瞬时时间，精确到毫秒，可用于记录时间戳

java 8支持通过时区id，时区偏移量来获取时间

在实际开发中我们关注的有以下几个方面的时间：
1. 时间戳，既有毫秒，也有秒，秒主要是PHP等服务返回的标准时间戳
2. Date，不要忘了数据库对应的实体中，使用的时间对象仍是Date
3. 待格式化的字符串，很常见的需求，将字符串解析为时间，或是将时间格式化为文本
这几个方面时间的互相转换也需要关注

# API
## 获取时间
### now
now方法用于获取日期类的当前值，例如LocalDate获取当前日期，LocalTime获取当前时间的时分秒等信息
### 获取年月日
根据常识，仅LocalDate, LocalDateTime可以获取年月日，分别由getYear,getMonthValue,getDayOfMonth获取，时分秒的获取方式同理LocalDateTime
### 获取自1970-01-01T00:00:00的毫秒数，秒数，天数
不用死记硬背，只要想清楚这几个类分别代表了什么类型的时间即可推断出api
```java
Instant.now().toEpochMilli();
LocalDateTime.now().toEpochSecond(ZoneOffset.ofHours(8));
LocalDate.now().toEpochDay();
```
ZoneOffset.ofHours(8)可以理解为北京时间位于东八区
### 获取前一天，一个月，一年等等
记住两个单词即可，minus表示减，plus表示加
至于你想加减些什么，首先要确定要加减的单位是什么，比如分钟，那肯定是在LocalDateTime，LocalTime里找，加年，加月同理，剩下的api就不啰嗦了，授人以鱼不如授人以渔，读者有兴趣自己探索。

## 时间转换
### 文本转时间
parse用于处理从文本到日期的转换，根据DateTimeFormatter的格式解析成日期
```java
LocalDate.parse("2019-01-11", DateTimeFormatter.ofPattern("yyyy-MM-dd"))
```
yyyy-MM-dd是默认的格式，可以省略第二个参数，类似的HH:mm:ss在转换为时分秒时也可以省略


### 毫秒时间戳转时间
注意两点：
1. Instant用于表示瞬时值，它和秒，毫秒是相关联的，再将Instant转换为LocalDateTime
```java
long time = 1548154964271L;
Instant instant = Instant.ofEpochMilli(time);
LocalDateTime dateTime = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
```
2. 需要判断时间戳是毫秒还是秒，PHP等语言只能精确到秒，请求这类接口时需注意
给出一个转换方法示例
```java
public static final long MAX_TIME_STAMP = 10000_000_000L;

public static final long MIN_TIME_STAMP = 1000_000_000L;

private static Instant transToInstant(long time) {
    if (time < MAX_TIME_STAMP && time > MIN_TIME_STAMP) {
        return Instant.ofEpochSecond(time);
    }
    if (time < MAX_TIME_STAMP * 1000 && time > MIN_TIME_STAMP * 1000){
        return Instant.ofEpochMilli(time);
    }
    throw new IllegalArgumentException("illegal time value");
}
```

### 与Date类的转换
记住一点，旧版的Date与java 8中的日期类的转换桥梁是Instant
```
Date date = Date.from(LocalDateTime.now().atZone(ZoneId.systemDefault()).toInstant());

LocalDateTime dateTime = date.toInstant().atZone(ZoneId.systemDefault()).toLocalDateTime();
```

## 获取时间段
Period和ChronoUnit都可以做到计算时间段，下面以计算两个时间时间的天数为例
```java
Period period = Period.between(LocalDate.of(2019, 1, 19), LocalDate.of(2019, 2, 10));
long days = ChronoUnit.DAYS.between(LocalDate.of(2019, 1, 19), LocalDate.of(2019, 2, 10));
```
# 结束语
java 8时间类的使用写到这里告一段落，关于时区的使用，也见缝插针的介绍了下，写这篇文章对笔者最大的挑战是表达能力，api很多，我想表达的是有规律的使用api，而不是死记硬背，最后分享一个自己写的DateUtils

# DateUtils
```java
/**
 * 舍弃对Calendar，joda-time的依赖，用java 8重写
 *
 * @author yangjie
 * @createTime 2019/1/17
 * @since 1.0.0
 */
public class DateUtils {

    public static LocalDate today = LocalDate.now();

    public static LocalDateTime now = LocalDateTime.now();

    public static final String GENERAL_PATTERN = "yyyy-MM-dd HH:mm:ss";

    public static final String GENERAL_DATE_PATTERN = "yyyy-MM-dd";

    public static final long MAX_TIME_STAMP = 10000_000_000L;

    public static final long MIN_TIME_STAMP = 1000_000_000L;

    /***
     * 分别获取年月日的值
     * @return
     */
    public static <L, M, R> Triple<Integer, Integer, Integer> getYearMonthDay() {
        return ImmutableTriple.of(today.getYear(), today.getMonthValue(), today.getDayOfMonth());
    }

    /**
     * 获取日期文本
     */
    public static String getDateText() {
        return today.format(DateTimeFormatter.ofPattern(GENERAL_DATE_PATTERN));
    }

    /**
     * 获取时间本文
     */
    public static String getDateTimeText() {
        return now.format(DateTimeFormatter.ofPattern(GENERAL_PATTERN));
    }

    /**
     * 根据毫秒数获取时间文本
     */
    public static String getDateTimeText(long time) {
        return now.format(DateTimeFormatter.ofPattern(GENERAL_PATTERN));
    }

    /**
     * 给定日期是否是今日
     */
    public static boolean isToday(String time) {
        return LocalDate.parse(time).isEqual(today);
    }

    /**
     * 当前时间戳是否是今天
     */
    public static boolean isToday(long time) {
        LocalDate date = LocalDateTime.ofInstant(transToInstant(time), ZoneId.systemDefault()).toLocalDate();
        return date.isEqual(today);
    }

    /**
     * 获取n天之前至今的时间段
     */
    public static Pair<Long, Long> getDaysAgoTime(Integer days) {
        LocalDate localDate = LocalDate.now().minusDays(days);
        long begin = localDate.atTime(0, 0, 0).toEpochSecond(ZoneOffset.ofHours(8));
        long end = localDate.atTime(23, 59, 59).toEpochSecond(ZoneOffset.ofHours(8));
        return ImmutablePair.of(begin, end);
    }

    /**
     * 获取两个时间段区间 00:00:00 - 23:59:59
     */
    public static Pair<Long, Long> getDurationPair(String beginStr, String endStr) {
        long begin = LocalDate.parse(beginStr, DateTimeFormatter.ofPattern(GENERAL_DATE_PATTERN)).atTime(0, 0, 0).toEpochSecond(ZoneOffset.ofHours(8));
        long end = LocalDate.parse(endStr, DateTimeFormatter.ofPattern(GENERAL_DATE_PATTERN)).atTime(23, 59, 59).toEpochSecond(ZoneOffset.ofHours(8));
        return ImmutablePair.of(begin, end);
    }

    /**
     * 获取两个时间段之间的天数
     */
    public static long getBetweenDays(long start, long end) {
        return ChronoUnit
                .DAYS
                .between(
                        LocalDateTime.ofInstant(transToInstant(start), ZoneId.systemDefault()),
                        LocalDateTime.ofInstant(transToInstant(end), ZoneId.systemDefault())
                );
    }

    /**
     * 获取两个时间段之间的天数
     */
    public static long getBetweenDays(String start, String end) {
        return ChronoUnit.DAYS.between(LocalDate.parse(start), LocalDate.parse(end));
    }

    /**
     * 获取两个时间段之间的所有日期
     */
    public static List<String> getBetweenDates(String start, String end) {
        List<String> list = new ArrayList<>();

        LocalDate startDate = LocalDate.parse(start);
        LocalDate endDate = LocalDate.parse(end);

        long distance = ChronoUnit.DAYS.between(startDate, endDate);

        if (distance < 1) {
            return list;
        }

        if (distance == 0) {
            list.add(start);
            return list;
        }

        Stream.iterate(startDate, d -> d.plusDays(1)).limit(distance + 1).forEach(f -> {
            list.add(f.toString());
        });
        return list;
    }
}
```






