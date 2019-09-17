[TOC]

# 深入学习Android源码（AOSP）

## Android平台架构

> [官网关于平台架构的说明](https://developer.android.com/guide/platform)

首先先从整体上对Android整个平台架构有个初步认识（图来自官网）：

![ The Android software stack](https://developer.android.com/guide/platform/images/android-stack_2x.png)

## 目标和分析方法

### 目标

根据平台架构，可以发现平时的应用开发工作基本是建立在`Java API Framework`之上，所以一开始最应该弄懂的就是`framework`这一层，但因为跨进程`Binder(IPC)`作为非常基础和核心的部分，也会在开始阶段由浅到深的去理解。

### 分析方法

基于已有的一些分析博客，同时带着自己的疑问去分析，并记录分析过程。

## 前期准备工作

- [x] 1. 下载AOSP源码（源码版本：android-9.0.0-r40），并成功编译，运行模拟器
- [x] 2. 源码导入到`Android Studio`以便阅读，以及断点调试

> 关于如何下载编译并运行模拟器，查看官网和其它博客说明。以下记录几个关键点：
>
> - 同步代码时使用网上提供好的重试脚本
> - 自己用的Mac电脑编译时选择`aosp_x86-eng`，成功运行模拟器
> - 导入`Android Studio`阅读源码，一开始可以先`exclude`掉所有模块，然后再把需要的模块的`java`目录标记为`Sources Root`，例如要导入`frameworks`，不要直接把整个目录标记为源码，而是把`frameworks/base/core/java`标记为源码，否则会出现代码无法正确跳转的问题

## 成果

分析过程形成的关键结果记录成文章。

### framework

> 目录结构对应于上文架构图，有些文章内容会涉及到多个部分。

#### Activity

- [Activity.setContentView()源码分析](./doc/Activity.setContentView()源码分析.md)

#### Window

- [寻找Activity布局树的根](./doc/寻找Activity布局树的根.md)

### Linux Kernel

#### Binder(IPC)

-  [Binder学习资料汇总](doc/Binder学习资料汇总.md) 