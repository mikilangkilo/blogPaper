---
title: 安卓基础-Android系统信息和安全机制
date: 2018-01-25 11:51:19
tag : "activity"
category : "安卓基础"
description : "本文是学习Android持久化存储的笔记"
---

# Android系统信息获取

获取系统的配置信息，通常从build和systemproperty两个方面获取

1. Android.os.Build

该类里面的信息非常丰富，包含了系统编译时的大量设备、配置信息

2. SystemProperty

该类包含了许多系统配置属性值和参数，有一些和build是相同的。

3. Android系统信息实例

```
	String board = Build.BOARD;
	String brand = Build.BRAND;

	String os_version = System.getProperty("os.version");
	String os_name = System.getProperty("os.name");
```

# Android Apk应用信息获取之PackageManager

PM主宰着应用的包管理

1. ActivityInfo: 封装了在Mainifest文件中<activity></activity>和<receiver></receiver>之间的所有信息，包括name，icon, label, launchmod等

2. ServiceInfo: ServiceInfo与ActivityInfo类似，封装了<service></service>之间的所有信息

3. ApplicationInfo: 封装了 <application></application>之间的信息，特别的是，applicationinfo包含了很多flag，通过这些flag，可以很方便的判断应用的类型

4. PackageInfo: 封装了所有的activity，service等信息

5. ResolveInfo: 封装的是包含<intent>信息的上一级信息，所以可以返回Activityinfo, ServiceInfo等包含<intent>的信息，可以用来找到含有特定intent条件的包

# Android Apk应用信息获取之ActivityMananger

AM可以获取正在运行的应用程序信息。

1. AcitivtyManager.MemoryInfo

全局内存信息，availMem是系统可用内存，totalMem是总内存，threshold是低内存的阀值，lowMemory是检查是否处于低内存。

2. Debug.MemoryInfo

用于获取统计进程下的内存信息。数据是由dvm虚拟机提供的。

3. RunningAppProcessInfo

运行进程的信息

4. RunningServiceInfo

用于封装运行的服务信息。

