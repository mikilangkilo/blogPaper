---
title: "安卓基础-Rom编译"
date: 2017-12-21 16:24:02
tag : "Rom"
category : "安卓基础"
description : "本文主要是学习编译rom"
---

# AOSP下载

下载有两种方式，第一种是直接init repo，然后repo sync，不过在大陆几乎无法成功。

因此使用第二种方式

## 第一步下载初始化包

```
wget -c https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar
```

47个g。。。。。。mac的记得要装一个移动硬盘

## 第二步解压

```
tar xf aosp-latest.tar
```

## 第三步repoinit

注意要进入解压的目录内
```
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest
```
如果出现repo command not found，调用以下命令

```
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc 
export PATH=$PATH:$HOME/bin 
```

之后repo就有了

## 第四步repo sync

```
repo sync
```

这一步也十分可能卡住，建议使用

```
repo sync -f -j8
```

同时这一步极有可能出现checkout error的错误，不过会提示出错误的路径和checkout的hash，因此需要手动到对应目录下执行stash和checkout指令

无尽的等待和重试...

# AOSP编译

编译的过程和以前framework全编的过程基本相同

根目录下执行
```
make -j8
```

又是无尽的等待....


