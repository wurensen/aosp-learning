[TOC]

# Android进程学习资料汇总

## Linux进程和线程

在`Linux`操作系统中，线程的本质也是进程，所有的线程都当作进程来实现。对于一个进程来说，相当于它含有一个线程，就是它自身；对于多线程来说，原本的进程成为“主线程”，它们在一起组成一个线程组。

`Linux`创建进程的核心函数是`fork()`，所以需要先理解该函数的作用，以便去理解`Android`的进程创建。

> 关于Linux进程和线程的资料：
>
> - [Linux进程与线程的区别](https://my.oschina.net/cnyinlinux/blog/422207)

### fork机制

`fork`过程复制资源包括代码段，数据段，堆，栈。fork调用者所在进程便是父进程，新创建的进程便是子进程；在`fork`调用结束，从内核返回两次，一次继续执行父进程（同时返回子进程pid），一次进入执行子进程（同时返回pid为0）。

> 关于`fork`机制的资料：
>
> - [linux中fork（）函数详解（原创！！实例讲解）](https://blog.csdn.net/jason314/article/details/5640969)

理解了`fork`的过程，就比较好理解进程创建。

## Zygote进程

`Zygote`进程是`Android`系统的第一个`Java`进程(即虚拟机进程)，`System Server`进程（即：`system_server`）是`Zygote`孵化的第一个进程，所有的App进程都是由`Zygote`进程`fork`生成的。而`Zygote`进程是由`Linux`系统中用户空间的第一个进程`init`进程（**pid=1**）`fork`而来。

所以想了解App进程的创建过程，有必要先了解`Zygote`进程的创建过程、`system_server`进程的创建过程以及`Zygote`进程是如何通过`LocalSocket`机制接收消息，并`fork`出`App`进程的。

- [Android P (9.0) 之Zygote进程源码分析](https://blog.csdn.net/wangzaieee/article/details/85003806)

## SystemServer进程

`SystemServer`进程（进程名：`system_server`）是在`Zygote`进程还未进入`LocalSocket`监听前`fork`出来的，它的初始化过程包含了：

1. 启动`Binder`线程池，这样就可以与其他进程进行`Binder`跨进程通信
2. 创建`SystemServiceManager`，，它用来对系统服务进行创建、启动和生命周期管理
3. 多个系统服务的启动：引导服务、核心服务、其他服务；这些服务包括核心的`ActivityManagerService`、`PowerManagerService`，以及`WindowManagerService`等

- [Android系统启动流之SystemServer进程启动](https://jsonchao.github.io/2019/03/03/Android系统启动流之SystemServer进程启动/)

## 应用进程

`Android`的应用进程都是由`Zygote`进程`fork`而来，要创建应用进程，首先是通过`Binder`方式调用`system_server`进程的相关函数，然后通过`socket`与`zygote`进程产生通讯，最终创建出应用进程，并执行`ActivityThread`的` main`函数。

- [理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/)

## 系统启动架构图

![系统启动架构图](http://gityuan.com/images/android-arch/android-boot.jpg)