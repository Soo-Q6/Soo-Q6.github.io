---
title: Java的类加载机制
# image: /assets/img/blog/...
description: >
  
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
---

虚拟机把描述类的数据从class文件加载到内存，并对数据进行校验，转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制

## 加载

* 通过一个类的全限定名来获取此类的二进制字节流
* 将这个字节流所代表的静态存储结构转换为方法区的运行时数据
* 内存中生成一个Class对象(存放方法区)，作为方法区这个类的各种数据的访问入口

## 验证
验证大概分为文件格式验证，元数据验证，字节码验证，符号引用验证(发生在将符号引用转换为直接引用的过程)
## 准备
分配类内存和设置变量的初始值
## 解析
将常量池内的符号引用转换为直接引用的过程(这里是**静态解析**，还有一个在程序运行时的**动态解析**)
## 初始化
根据程序进行初始化
## 使用

## 卸载