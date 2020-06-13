---
title: JNI程序规范和指南1——简介
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---
JNI是Java平台提供的一个强大的功能，使得Java应用可以重用C和C++写的本地代码(native code)。这系列文章既是编程指南，也是JNI手册，本书包含三个部分：<br>

* 第二章通过一个例子来介绍JNI，适用于JNI初学者.
* 第三章到第十章是一些JNI特征的概述，我会举很多简单的例子来介绍JNI的各个特征，这些特征在JNI中是很重要的.
* 第十一章到第十三章给出明确的JNI类型和方法的规范，这一部分你可以认为是一个JNI手册.<br>

看到这里，首先默认你有一定的Java，C和C++的基础。

## Java平台和主机环境(Host Environment)
由于本文涉及到Java，C和C++写的应用，所以首先明确一下这些语言的编程环境。<br>
Java平台是一个包含Java虚拟机和Java API的编程环境，Java应用会被编译为一个独立于机器的，可以被任一Java平台执行的二进制文件；主机环境(Host Environment)指的是本地主机的操作系统环境，有着自己的**本地库**以及**CPU指令集**。本地应用(Native Application)是指使用C/C++编写的，能够被编译为依赖于主机环境的可执行文件，链接着本地库，一般只能在本机运行。Java平台运行在本机环境之上，并提供给一些不依赖于主机环境的特性。
## JNI的定位
JNI提供了在Java平台上使用本地C/C++代码的能力，作为Java虚拟机实现的一部分，JNI是一个**双向**的接口，既允许Java调用C/C++，也允许C/C++有调用Java的能力, 如下图：

![role of JNI]({{site.data.strings.blog_url}}JNI1-1.png)<br>

JNI支持一下两种方式和本地代码交互：
* 可以使用JNI来“实现”基于本地库的方法(native method)以供Java应用使用, Java应用可以像调用Java API一样调用JNI实现的方法(Java中带有`native`声明的函数)，但实际上这些方法是通过C/C++在本地实现的。
* JNI支持invocation interface, 借此你可以将一个JVM嵌入到本地应用之中。本地应用可以“实现”一个JVM，然后通过invocation interface执行Java实现的组件。比如一个C实现的浏览器可以通过嵌入一个JVM来执行网上下载的applets。<br>

## 使用JNI的风险
一旦你使用了JNI，你就丧失了Java平台的两个优点：
1. 基于JNI的Java应用不再支持任一平台，你需要重新编译基于本地方法实现的代码
2. Java是类型安全的而C/C++不是，本地代码的不当使用会导致整个程序的崩溃<br>

一个通用的规则是，尽量将本地代码集中在少数的几个类，减少Java和C/C++的耦合

## 什么时候使用JNI
使用JNI之前要慎重考虑是否有其他的解决方案。<br>
这里有一些别的方法提供了Java和本地代码的交互能力：
* TCP/IP连接或者进程间通信(IPC)
* 通过JDBC API与本地数据库进行连接
* JAVA程序可以使用分布式对象技术(distributed object technologies)，如JAVA IDL API<br>

上面提到的几种方法的共性是，Java应用和本地应用是处于不同进程之间的，但有时候你又不得不在同一个进程内使用Java和C/C++:
* Java API无法提供与主机环境相关的方法，比如Java平台不支持某个特殊的文件操作而通过其他进程进行文件操作效率很低
* 你想直接加载一个本地库而不想再花精力去实现一遍
* 跨多个进程的应用程序是很占系统资源的，如果将本地库加载到同一个进程内，会节省不少资源(使用多线程?)
* java应用中有不少对计算能力要求很高的代码，比如图形渲染等，需要使用到效率更高的底层语言。<br>

## JNI的进化史
JDK 1.0中包含了一个本地接口允许Java代码调用C/C++，很多Java库(java.lang, java.io, java.net)是基于这个本地接口来访问本地环境的。但是JDK 1.0的本地接口有两个主要的问题：<br>
1. 本地代码以访问C结构体成员的方式访问对象中的字段，但是JVM规范中没有定义对象在内存中的存放方式，一旦本地代码无法识别jvm中对象存放方式，你就需要中心编译本地代码
2. 本地方法可以直接控制JVM对象的指针，所以JVM需要一个保守的垃圾回收机制，其他支持更先进的垃圾回收算法的JVM是无法兼容JDK1.0的本地接口的。<br>

JNI就是用来解决以上两个问题的，它可以在任何平台上的JVM中使用：
* 每一个虚拟机都支持大量的本地代码
* 开发工具供应商不用处理各种不同的本地接口
* 开发者设计的本地代码可以在不同的JVM中使用<br>

从JDK1.1开始支持JNI。

***
* 第一部分，简介和JNI入门

[JNI程序规范和指南1——简介](/blog/2019-09-27-JNI-guides-and-specifications-1/)<br>
[JNI程序规范和指南2——一个简单的例子](/blog/2019-09-28-JNI-guides-and-specifications-2)
* 第二部分，JNI指南

[JNI程序规范和指南3——基本类型, 字符串和数组](/blog/2019-09-29-JNI-guides-and-specifications-3)<br>
[JNI程序规范和指南4——字段和方法](/blog/2019-09-30-JNI-guides-and-specifications-4)<br>
[JNI程序规范和指南5——JNI中的局部引用和全局引用](/blog/2019-10-02-JNI-guides-and-specifications-5)<br>
[JNI程序规范和指南6——异常](/blog/2019-10-09-JNI-guides-and-specifications-6)<br>
* 第三部分，JNI规范