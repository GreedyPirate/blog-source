---
title: linux命令手记
date: 2018-08-27 17:03:03
categories: Linux
tags:
	- [日志]
toc: true
comments: true
---

## 从后往前查看日志

1. less 文件名
2. shift+g跳转到末尾，向上滑动

#### 使用场景

首先不推荐cat,vim等命令,大日志文件容易导致内存不足，线上排查问题时容易引起服务崩溃

有时想要查看最后五分钟内的日志，tail命令指定行数也可以大致做到，但是行数不好指定时，less会很方便

## 查找进程命令如何排除自带的grep

这个技巧常用在编写shell脚本时，希望查找到某个进程的pid，但是grep命令本身也会产生一条数据，因此需要排除
例如查找xxx进程时
```bash
ps -ef|grep xxx |grep -v 'grep'
```

## 查看前十个最占内存的应用

```bash
ps aux|head -1;ps aux|grep -v PID|sort -rn -k +4|head
```

## 查看前十个最占CPU的应用
```bash
ps aux|grep -v PID|sort -rn -k 3| head | awk 'BEGIN{print "USER PID %CPU %MEM VSZ RSS STAT"} {print $1,$2,$3,$4,$5,$6,$8}'

```

## 按端口终止进程
```bash
#!/bin/sh
PORT=2181
PID=`lsof -i:${PORT} |grep -v PID |awk '{print $2}'`
if [ ${PID} ]; then
        echo "kill pid : ${PID}"
        kill ${PID}
else
        echo "could not find process with port:${PORT}"
fi
```

## 生成UUID

uuidgen命令

```bash
uuidgen
```
结果

```bash
d4586ba5-22da-42e5-9662-acad5942988d
```

## 启动shell脚本

编写shell脚本之后，可以通过`chmod +x `的方式，然后启动，不过用`sh 脚本名`的方式更加简洁，目前没有对比二者的优劣

## 查找占用磁盘空间的文件

一层一层目录查找最占磁盘空间的文件夹

```bash
du -h --max-depth=1
```

## top命令技巧

shift+p: 按CPU使用率降序排序，用户查找CPU使用率最高的进程
shift+m: 按内存使用率降序排序
shift+h: 显示线程占用cpu情况

top -p <pid> -H ：查看进程中各线程详情


## netstat命令技巧

按pid统计连接数
```bash
netstat -natp|awk 'print $7'|uniq -c|sort -rn
```

## awk入门
test.txt
red,10,jay
black,20,tom
yellow,30,jim

awk -F "," '$3>15 {print $1,$2}' test.txt
持续积累中 ......


