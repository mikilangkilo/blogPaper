---
title: "java基础-深入理解javap"
date: 2020-04-27 16:11:29
tag : "javaP"
category : "java基础"
description : "深入理解javap"

---

javap一般用到javap -c -v -l 等参数，但是一般cv之间切换来看。大致的方法懂一些，百度能看出来干了啥，但从来也没总结过。

这次想通过字节码来理解一下锁的实现，觉得还是首先仔细学习一下机器码部分操作，梳理一下机器码的知识点

# java文件，class文件，机器码文件

以.java为后缀的文件就是java源文件，这个源文件是给程序员看的，未经过编译的初始代码

以.class为后缀的文件是java源文件编译之后的文件，这个是经过编译器编译过的，去掉了平台特性的代码，对.java文件使用javac即可
其中.class文件就是编译后的文件

机器码则是对.class文件进行解释之后的代码。

# 解释过程

java解释运行的过程事实上以前总结jit和aot的时候已经总结过了。

期间我产生了一个错觉，就是.class解释之后编程了.so，事实上是完全错误的，jit是解释成一段一段的机器码进行跑，aot是解释成aot文件，然后一段一段跑，和so没有任何关系。

# javap

```
用法: javap <options> <classes>
其中, 可能的选项包括:
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
```





