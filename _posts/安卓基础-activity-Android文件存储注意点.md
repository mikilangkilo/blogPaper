---
title: "安卓基础-Android文件存储注意点"
date: 2020-07-07T16:42:30+08:00
tag : "存储"
category : "安卓基础"
description : "本文是学习Android持久化存储的笔记"
---

文件存储主要是用到的内部存储，外部存储需要读写权限，大多数时候应用的缓存，图片之类的，最好还是放到内部存储中。

把内部存储的细节捋一下

# 内部存储空间有限

使用内部存储占用的空间会被记录到应用的占用存储之中，而手机内部存储的空间是有限的，超过了阙值之后就无法存储。
比较好识别的是，内部存储就是在data/data/包名/下面的内容

# 内部存储方法api

## Environment.getDataDirectory()

Vivo:/data
Huawei:/data
Xiaomi:/data

这个目录下存放了lib，shared_pref，files,cache,总而言之这是内部存储的总目录，创建文件的时候可以在里面创建目录存放，但是不能乱删

## context.getFilesDir().getAbsolutePath()

Vivo:/data/user/0/com.ximalaya.ting.Android.car/files
Huawei:/data/user/0/com.ximalaya.ting.Android.car/files
Xiaomi:/data/user/0/com.ximalaya.ting.Android.car/files

这个目录也是存放一些文件的，官方推荐是使用这个目录存放一些不想删除的文件，或者只能应用自己删除的文件。像MMKV就是放在里面，Xlog也是

这个里面放的是一些三方框架的

## context.getCacheDir().getAbsolutePath()

Vivo:/data/user/0/com.ximalaya.ting.Android.car/cache
Huawei:/data/user/0/com.ximalaya.ting.Android.car/cache
Xiaomi:/data/user/0/com.ximalaya.ting.Android.car/cache

这个起名就是缓存目录，这个就是官方推荐的缓存目录，放在缓存目录的文件一般应该是临时存放的，同时APP需要给出提示缓存有多少，以及清除缓存的方式。
cache目录是系统自动创建的，但是系统不会自动回收，开发者理应做好内存临界时旧文件的删除操作。

# 设置中的存储大小

设置中的APP存储，普遍分为总计/应用/数据/缓存

## 总计

在app安装并启动之后

小米的总计是59.92m，华为是19.2M，VIVO是30.69M

这个总计的计算规则不懂，应该每家都不一样

## 应用

应用应该是APP安装后占用的大小，这个华为和VIVO是一样，都是安卓10的平台，app安装都是18.62m，而小米是8.0的，就大很多，打到了33.31M。

## 数据

同样是才打开，VIVO的数据是最小的，只有600K，而华为是7M，小米是26.6M

数据这个在小米称作用户数据，应该是算上了用户的账户信息。

## 缓存

缓存基本都是相同的，6.9M的样子

## 清空缓存

清空缓存之后，cache目录里面的东西会消失

## 清空全部数据

清空全部数据之后，整个包文件夹下面会被删的还剩lib，这样子等于重装了。

# 总结

存文件存到cache里面，才是清除缓存的唯一功效啊！



