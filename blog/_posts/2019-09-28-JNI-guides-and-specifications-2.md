---
title: (译)JNI程序规范和指南2——一个简单的例子
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---
本文通过一个简单的例子来介绍如何使用JNI：一个Java程序调用C代码输出hello world。<br>


## 概述
下图描述了Java程序调用C代码输出hello world的过程 <br>

![Steps in Writing and Running the “Hello World” Program]({{site.data.strings.blog_url}}JNI2-1.png)<br>

这个过程主要包含以下几个步骤: <br>

1. 创建一个HelloWorld.java文件声明了本地方法(native method)。
2. 使用`javac`编译Java源码，生成`.class`文件。
3. 使用`javah -jni`生成C头文件，这个文件包含了这个本地方法。(native method)的函数原型。
4. 在HelloWorld.c文件中实现本地方法(native method)。
5. 使用本地C编译器编译生成本地库文件(.dll后者.so)。
6. 运行Java程序，`.class`和`.dll`/`.so`运行时加载。

## 声明本地方法(native method)
首先java程序定义了一个HelloWorld的类，包含了一个本地方法`print`
```java
import java.lang.*;
class HelloWorld{
    private native void print();
    public static void main(String args[]) {
        new HelloWorld().print();
    }
    static{
        System.loadLibrary("HelloWorld");
    }
}
```
HelloWorld这个类首先定义了本地方法`print`，然后`main`函数中实例化HelloWorld并调用`print`。最后通过`System.loadLibrary`加载对应的本地库。<br>

声明本地方法必须使用关键字`native`，然后会在C中实现这个函数。在本地方法被调用之前，必须确保本地库已经被加载，这里我们使用static的方式初始化加载，确保了在调用`print`之前本地库已被加载。<br>

`System.loadLibrary`需要一个库名字，为了`System.loadLibrary("HelloWorld")`能够成功执行，需要在Windows创建`HelloWorld.dll`或者在Linux中创建`libHelloWorld.so`。

## 编译HelloWorld类
使用`javac HelloWorld.java`编译生成`.class`文件

## 创建本地方法的C头文件
使用`javah -jni HelloWorld`生成一个JNI风格的头文件, 然后在HelloWorld.h中会有对应的`print`函数原型:
```c
JNIEXPORT void JNICALL Java_HelloWorld_print
  (JNIEnv *, jobject);
```
先不管`JNIEXPORT`和`JNICALL`两个宏。你可能注意到本地方法的C实现有两个形参即使Java代码中并没有任何形参的定义，C的本地方法实现的第一个形参是`JNIEnv`指针，这里你可以简单的理解为他是一个Java环境的实现，提供了一些接口；第二个形参是一个HelloWorld对象的引用(类似于C++的`this`指针)。后面会继续讲解他们的用法。

## 本地方法的实现
你需要根据生成的头文件来实现本地方法:
```c
#include <jni.h>
#include <stdio.h>
#include "HelloWorld.h"

JNIEXPORT void JNICALL Java_HelloWorld_print
  (JNIEnv *env, jobject obj){
      printf("Hello World in C\n");
      return;
  }
```

## 编译C源文件并创建一个本地库
不同的操作系统支持不同的库文件，比如Linux支持的.so：
```shell
cc -G -I/java/include -I/java/include/solaris HelloWorld.c -o libHelloWorld.so
```
Windows则需要生成.dll，你需要进入vs的命令行工具(比如VS2015 x64 本机命令工具提示符)执行下列命令(前提是你安装了visual studio和Java环境)：
```shell
cl -I"%JAVA_HOME%\include" -I"%JAVA_HOME%\include\win32" -LD HelloWorld.c -FeHelloWorld.dll  
```
*上面的编译方法中Linux是书中的命令，没有验证过是否能够执行；Windows书中的命令执行不成功，以上是根据我的使用环境修改过后的命令。*

## 运行Java程序
完成上述步骤之后，你就可以执行Java程序了:
```
java HelloWorld
```
输出的接口是:
```
Hello World in C
```

***
* 第一部分，简介和JNI入门

[(译)JNI程序规范和指南1——简介](/blog/2019-09-27-JNI-guides-and-specifications-1/)<br>
[(译)JNI程序规范和指南2——一个简单的例子](/blog/2019-09-28-JNI-guides-and-specifications-2)
* 第二部分，JNI指南

[(译)JNI程序规范和指南3——基本类型, 字符串和数组](/blog/2019-09-29-JNI-guides-and-specifications-3)<br>
[(译)JNI程序规范和指南4——字段和方法](/blog/2019-09-30-JNI-guides-and-specifications-4)<br>
[(译)JNI程序规范和指南5——JNI中的局部引用和全局引用](/blog/2019-10-02-JNI-guides-and-specifications-5)<br>
[(译)JNI程序规范和指南6——异常](/blog/2019-10-09-JNI-guides-and-specifications-6)<br>
* 第三部分，JNI规范