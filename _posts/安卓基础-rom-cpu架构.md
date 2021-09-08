---
title: "安卓基础-Android的cpu架构学习"
date: 2019-05-08T11:42:04+08:00
tag : "Rom"
category : "安卓基础"
description : "本文主要是学习Android的cpu架构"
---

以往听过arm架构和x86架构，在jni和选用so库的时候需要注意一下，但这次项目转型flutter，遇到的flutterlibso的问题，也是关于arm架构的问题。

# 架构是什么

abi application binary interface

abi是比api更接近硬件的一层接口，规定的是二进制代码之间的规则。

从Androidflutter的角度来讲，就是当flutter生成了一个libflutter.so的时候，这个so就是被用于调用的二进制代码。

但是libflutter.so在编译的时候需要决定编译出来的是armv7，x86，还是arm等，这个不同的架构，在Android中会进行区分。如果app是基于armv7架构的，那么只会调用armv7架构的libflutter.so。

# arm架构

arm处理器一般广泛用于通信，全称为精简指令集RISC处理器架构。
arm的优点在于效率，其处理任务相对固定的应用场合比较高效，但是缺点在于处理综合型的任务比较低效

另外arm处理器的配置，类似于存储，内存等基本上出场就已经设置好，不容易更改

## arm64

arm64是使用64位处理器中集成的32位架构来运行32位程序，并不是完全意义上的64位

# x86架构

x86一般是电脑架构，易于扩展，且成本高，功耗也不低。

# 64位手机

64位手机是指完全的64位，包括Android虚拟机、处理器、系统、以及程序。

对Android来讲，需要在4.4以后的手机才是64位手机。因为dalvik和art虚拟机都是32位的

# Android分包

```
splits {
        abi {
            enable true
            reset()
            include 'x86'
            exclude 'armeabi', 'armeabi-v7a', "arm64-v8a"
            universalApk true
        }
}
```
每次打包都会生成include中写的包，一般情况下不配置，但是如果配置的话需要注意一下。

# ndk架构

使用ndk架构来配置的话，会将需要的so库全部打包到一个apk中，因此不需要通过分包。

```
ndk {
    abiFilters 'armeabi', 'armeabi-v7a', 'arm64-v8a'
}
```

这样的话三个包的架构都会打包到一个包下。

# libflutter.so could not found 问题

该问题发生的原因是，flutter在打包的时候生成的so库和手机的架构还有apk中指定的架构不同，会出现找不到的情况

将ndk设置为

```
ndk {
                abiFilters "armeabi-v7a", "armeabi"
            }

```

同时打aar的时候使用
```
flutter build apk --target-platform=Android-arm
```

这样会直接生成一个32位的arm架构aar，这样可以顺利解决该问题。