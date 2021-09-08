---
title: jvm方法调用机制
date: 2019-02-23 20:05:58
tags: "方法调用机制"
category: "java基础"
description: "本文是学习jvm方法调用机制相关"
---

# 方法栈

JVM内存模型中，有一个栈结构，就是java方法栈，栈里面存放的一个个实体类称为栈帧。
每个栈帧都包括了局部变量表，操作数栈，动态连接，方法返回地址和一些额外的附加信息。

# 局部变量表

局部变量表用于存放方法参数和方法内部定义的局部变量，局部变量表的容量一般以32位为最小单位。

如果方法是非static方法，那局部变量表中第0位所以一般是实例对象的引用this。

# 操作数栈

JVM解析执行字节码是基于栈结构的，做算数运算时是通过操作数栈来进行的，在调用其他方法时是通过操作数栈来进行参数的传递。

# 方法调用过程

每一次方法调用指令之前，JVM先把方法被调用的对象引用压入操作数栈中，除了对象的引用之外，JVM还会把方法的参数依次压入操作数栈

在执行方法调用指令时，JVM会将函数参数和对象引用依次从操作数栈弹出，并新建一个栈帧，把对象引用和函数参数分别放入新栈帧的局部变量表

jvm把新栈帧push入虚拟机方法栈，并把pc指向函数的第一条待执行的指令。

# 方法调用的字节码指令

JVM提供了四种方法调用字节码指令

invokestatic:调用静态方法
invokespecial:调用实例构造器<init>方法，私有方法，实例构造器，父类方法
invokevirtual:调用所有的虚方法
invokeinterface:调用接口方法，会在运行时期在确定一个实现此接口的对象

```
public class Test {
    private void run() {
        List<String> list = new ArrayList<>(); // invokespecial 构造器调用
        list.add("a"); // invokeinterface 接口调用 
        ArrayList<String> arrayList = new ArrayList<>(); // invokespecial 构造器调用
        arrayList.add("b"); // invokevirtual 虚函数调用
    }
    public static void main(String[] args) {
        Test test = new Test(); // invokespecial 构造器调用
        test.run(); // invokespecial 私有函数调用
    }
}
```

字节码

```
public class Test {
  public Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  private void run();
    Code:
       0: new           #2                  // class java/util/ArrayList
       3: dup
       4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #4                  // String a
      11: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
      16: pop
      17: new           #2                  // class java/util/ArrayList
      20: dup
      21: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
      24: astore_2
      25: aload_2
      26: ldc           #6                  // String b
      28: invokevirtual #7                  // Method java/util/ArrayList.add:(Ljava/lang/Object;)Z
      31: pop
      32: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #8                  // class Test
       3: dup
       4: invokespecial #9                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: invokespecial #10                 // Method run:()V
      12: return
}
```

# 动态分派

当JVM遇到invokevirtual或者invokeinterface时，需要运行时根据方法的符号引用查到方法地址，步骤如下

- 在方法调用指令之前，需要将对象的引用压入操作数栈
- 在执行方法调用时，找到操作数栈顶的第一个元素所指向的对象实际类型，记做C
- 在类型C中找到与常量池中的描述符和方法名称都相符的方法，并校验访问权限，如果找到该方法并通过校验，则返回这个方法的引用。
- 否则，按照继承关系往上查找方法并校验访问权限
- 如果始终没找到方法，则抛出AbstractMethodError异常

# 虚函数表

java通过虚函数表实现多态。

方法表中包含了所有方法的入口地址，继承父类的方法在最前面，之后是接口方法，最后是自定义方法。
如果子类重写了父类的方法，那么地址会是子类方法的地址。否则则会指向父类的地址。

# invokevirtual 和 invokeinterface的区别

由于虚函数在编译时就可以确定offset，而实现了接口类型的类，直接使用接口方法的话，由于此时不确定其类型，会重新找一遍虚函数表，速度会降低。

因此在使用接口方法的时候，尽可能直接使用原有的类，而非使用接口类去转型。

不过个人认为在考虑架构的时候往往不可能照顾的那么仔细，此条理解即可。