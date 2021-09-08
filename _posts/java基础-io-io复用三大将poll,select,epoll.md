---
title: "Io复用三大将poll,select,epoll"
date: 2020-04-21T16:29:38+08:00
tags: "io"
category: "java基础"
description: "本文是学习io复用相关"
---

```
1.select/poll
老李去火车站买票，委托黄牛，然后每隔6小时电话黄牛询问，黄牛三天内买到票，然后老李去火车站交钱领票。
耗费：打电话
2.epoll
老李去火车站买票，委托黄牛，黄牛买到后即通知老李去领，然后老李去火车站交钱领票。
耗费：无需打电话



作者：凉拌姨妈好吃
链接：https://www.jianshu.com/p/6a6845464770
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

哈哈，通俗点讲就是这样啊。

# select

初次接触select还是学习okhttp之okio的时候学习的，依稀还记得channel，selector之类的名词

但事实上select作为同步io的一种实现方式，和okio的异步io还是有差别的

## select 构造

```
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

fd_set其实这是一个数组的宏定义，实际上是一long类型的数组，每一个数组元素都能与一打开的文件句柄(socket、文件、管道、设备等)建立联系，建立联系的工作由程序员完成，当调用select()时，由内核根据IO状态修改fd_set的内容，由此来通知执行了select()的进程哪个句柄可读。

fd_set可以通过一系列操作进行增加或者删除监听句柄的操作

void FD_ZERO(fd_set *fdset);
//清空集合
void FD_SET(int fd, fd_set *fdset);
//将一个给定的文件描述符加入集合之中
void FD_CLR(int fd, fd_set *fdset);
//将一个给定的文件描述符从集合中删除
int FD_ISSET(int fd, fd_set *fdset);
// 检查集合中指定的文件描述符是否可以读写

但是问题就是限制了最大句柄为1024，C环境下通过对结构体尾部参数增大内存可以更改句柄数量，但是并不安全，会带来宏失效等风险，因此有限制最大1024个句柄。
```

nfds：指定待测试的文件描述符个数，它的值是待测试的最大描述字加1。
readfds,writefds,exceptfds：指定了我们让内核测试读、写和异常条件的文件描述符
fd_set：为一个存放文件描述符的信息的结构体，可以通过下面的宏进行设置。
返回值：int 若有就绪描述符返回其数目，若超时则为0，若出错则为-1

## select 运行机制

（1）使用copy_from_user从用户空间拷贝fd_set到内核空间

（2）注册回调函数__pollwait

（3）遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）

（4）以tcp_poll为例，其核心实现就是__pollwait，也就是上面注册的回调函数。

（5）__pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同的设备有不同的等待队列，对于tcp_poll来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。

（6）poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。

（7）如果遍历完所有的fd，还没有返回一个可读写的mask掩码，则会调用schedule_timeout是调用select的进程（也就是current）进入睡眠。当设备驱动发生自身资源可读写后，会唤醒其等待队列上睡眠的进程。如果超过一定的超时时间（schedule_timeout指定），还是没人唤醒，则调用select的进程会重新被唤醒获得CPU，进而重新遍历fd，判断有没有就绪的fd。

（8）把fd_set从内核空间拷贝到用户空间。

## select 缺点

1、每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大

2、同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大

3、select支持的文件描述符数量太小了，默认是1024

## select优点

古老，因此每个平台都实现了，可以跨平台使用

# Poll

说实话我之前只知道epoll，不知道poll

## Poll的构造

```
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
typedef struct pollfd {
        int fd;                         // 需要被检测或选择的文件描述符
        short events;                   // 对文件描述符fd上感兴趣的事件
        short revents;                  // 文件描述符fd上当前实际发生的事件
} pollfd_t;
```

poll的构造对比select其实简便很多，由于使用了pollfd结构，因此没有fd_set的1024限制

fds为监听文件句柄集合

nfds为记录fds中描述符的总数量

返回值和select相同

## poll的机制

poll本质上和select无大区别，同select，先拷贝，后轮训，查得到处理，查不到休眠

## poll 缺点

poll的缺点和select很相同

1、每次都需要拷贝fds从用户态到内核态，而且poll无fds限制，因此可能会拷贝很大很大

2、仍然需要每次都遍历，时间复杂度仍然为O（n）

3、水平触发，fd不处理的话，不同于select会删除，而是会仍然放在里面，导致下一次还需要处理

4、我个人推测的，可能不是跨平台兼容

## poll优点

连接数（也就是文件描述符）没有限制（链表存储）

# Epoll

!!!大名鼎鼎，我们播放器就是用epoll来调的

## Epoll的构造

```
int epoll_create(int size);
创建一个epoll的句柄，size表示监听数目的大小。创建完句柄它会自动占用一个fd值，使用完epoll一定要记得close，不然fd会被消耗完。

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
是epoll的事件注册函数，和select不同的是select在监听的时候会告诉内核监听什么样的事件，而epoll必须在epoll_ctl先注册要监听的事件类型。
它的第一个参数返回epoll_creat的执行结果
第二个参数表示动作，用下面几个宏表示
EPOLL_CTL_ADD：注册新的fd到epfd中；
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；
第三参数为监听的fd,第四个参数是告诉内核要监听什么事

int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
等待事件的发生，类似于select的调用
```

```
epoll是Linux内核为处理大批量文件描述符而作了改进的poll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。

作者：似水牛年
链接：https://www.jianshu.com/p/397449cadc9a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

## Epoll的机制

epoll的机制描述起来较为复杂

1、调用epoll_create()建立一个epoll对象(在epoll文件系统中为这个句柄对象分配资源)

2、调用epoll_ctl向epoll对象中添加这个套接字

3、调用epoll_wait收集发生的事件的连接

同时epoll提供了边缘触发行为，可以使得一个事件仅仅通知一次

## Epoll的优点

1、创建对象的时候使用了mmap，而不需要频繁的拷贝

2、无句柄上限

3、复杂度O(1)，事件驱动，避免了傻瓜式的轮训

## 缺点

每次创建一个监听句柄，如果用poll的话就不需要创建了，而是在poll_fd中增加


# 总结

|       |Select | poll | epoll |
|-------|-------|------|-------|
|操作方式|遍历|遍历|回调|
|底层实现|数组|链表|哈希表|
|IO效率 |每次调用都进行线性遍历，时间复杂度为O(n)|每次调用都进行线性遍历，时间复杂度为O(n)|事件通知方式，每当fd就绪，系统注册的回调函数就会被调用，将就绪fd放到readyList里面，时间复杂度O(1)|
|最大连接数|1024（x86）或2048（x64）|无上限|无上限|
|fd拷贝|每次调用select，都需要把fd集合从用户态拷贝到内核态|每次调用poll，都需要把fd集合从用户态拷贝到内核态|调用epoll_ctl时拷贝进内核并保存，之后每次epoll_wait不拷贝|


# kqueue

