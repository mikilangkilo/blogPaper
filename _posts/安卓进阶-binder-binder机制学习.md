---
title: binder机制学习
date: 2018-10-18 22:42:23
tags: "binder"
category: "安卓进阶"
description : "本文主要是对binder进行学习和反思"
---
binder平时总听过，但是原理只知道是ipc，也就是进程间通信。但是真正的原理其实还是不理解。

# binder位置

binder介于framework和systemservice之间，属于让开发者来调用系统层接口的方法。

# ipc

ipc全称是inter-process communication,进程间通讯

ipc的方式有很多种，socket，共享内存，管道，消息队列

## socket

socket实现进程间通讯，是基于tcpip协议来实现的，一般实现tcp，如果是udp也是可以的，udp是两个socket但是无连接。
通过tcp实现ipc的原理
```
客户端程序通过socket发送一系列信息到传输层的tcp，然后往下传，通过网络层，网络接口层，然后在往上传到网络层，然后传到服务端的传输层tcp，然后由服务器的socket接收到，之后回传也是相同的
```

缺点是需传输效率较低，一般只用在不同机器，或者跨网络的通行

## 共享内存

共享内存的ipc，传输的对象一般都是可描述的，所以都用序列化，创建共享内存就使用memoryfile即可

这样构建的内存可以让其他进程共享

共享内存的优点在于无需复制，速度快，共享缓冲区直接附加到进程虚拟地址空间。缺点在于同步问题不好解决

## 消息队列

消息队列就是messagequeue
实现的思路
```
在进程a中创建一个message，讲这个message对象通过imessenger.send(message)方法传递到进程b中
send(message)会使用一个parcel对象对message对象编集，再将parcel对象传递到进程b中，然后解编集，得到一个和进程a中message对象内容一样的对象，在将message对象加入到b的消息队列里面，handler会处理它
```
消息队列的好处是比较方便，缺点是信息复制2次，有额外的cpu消耗，不是很适合频繁或者信息量大的通信

## 管道

管道是比较古老的通信方式，包括无名管道和有名管道，前者是父子进程间的通信，后者用于运行同一机器上的任意两个进程间的通信。实现方式是pipe，利用管道的有handler（此处需要研究一下handler的实现原理）

管道创建时分配了一个page大小的内存（page？）缓存区大小比较有限

# Binder

binder ipc属于c/s结构，client部分是用户代码，最终会调用binder driver的transact接口，binder driver会调用server

client:用户需要实现的代码，如aidl自动生成的接口类
binder driver：在内核层实现的driver
server：这个server就是service中onbind返回的ibinder对象

binder driver这块并不需要用户知道，server中会开启一个线程池（防止任务积压，也需要做好同步措施）去处理客户端调用

对于调用binder driver中的transact接口，客户端可以手动调用，也可以通过aidl的方式生成的代理类来调用，服务端可以继承binder对象，也可以继承aidl生成的接口类的stub对象

# 实现环节

//todo

