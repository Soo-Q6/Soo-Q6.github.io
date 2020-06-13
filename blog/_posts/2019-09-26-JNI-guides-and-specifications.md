---
title: JNI程序规范和指南0——前言
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---
最近在做一些播放器相关的工作，其中Android端使用FFmpeg库，需要通过JNI来调用，查看了不少文档，觉得**Java Native Interface-Programmer's Guide and Specification**一书讲得很清楚，所以想自己总结一下这本书的内容，以便日后查看。这个系列的每一篇文章对应着书的一个章节。<br>

这一个系列的文章主要内容就是JNI(JAVA native interface)，在以下的四种情况下你需要了解JNI:
1. 在Java应用中集成以前写的C代码
2. 需要整合JVM到现有的C/C++代码中
3. 实现一个JVM
4. 了解编程语言间相互操作的技术问题，特别是如何处理垃圾回收和多线程等功能

首先，这系列文章是写给开发者的。后续的文章会介绍如何快速的上手JNI，会讨论一些JNI的特性，已经如何有效的开发JNI程序。JNI初始发行实在1997年。<br>
其次，会介绍JNI特性的设计初衷。<br>
然后，文章的一部分是JNI的规范(JAVA2平台)。<br>
JNI的部分思想来源于Netscape的JRI(Java Runtime Interface)

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