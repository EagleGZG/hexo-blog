---
title: JVM 内存泄漏问题排查步骤
date: 2018-12-20
tags: [JVM]
---
# JVM 内存泄漏问题排查步骤

## 第一步
top

通过top命令来查找进程pid

## 第二步
top -H -p  pid

-H 指显示线程，-p 是指定进程。通过该命令可以查看该进程里面的线程的cpu占用情况。

## 第三步
printf "%x\n"  tid

将线程id转为16进制

## 第四步
jstack pid |grep tid -A 30

pid为第一步得到的进程的pid，tid为第三步得到的线程pid的十六进制数。

## 第五步
jmap -dump:format=b,file=文件名 [pid]

dump堆栈信息

## 其他
jmap -histo <pid>

该命令可以查看进程中对象的大小，可用于粗略判断是什么接口返回返回的什么对象发生了内存泄漏


