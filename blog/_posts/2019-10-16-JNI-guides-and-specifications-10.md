---
title: JNI程序规范和指南10——JNI陷阱
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---

本章主要总结一些在使用JNI时容易出现的错误。由于前面的文章大都已经介绍过，所以本文只是做总结性的介绍。

## 错误检测
最容易犯的错就是忘记去检查异常。JNI不依赖与任何本地异常处理机制，因此程序员需要在调用任意一个可能抛出异常的JNI函数时显式的检测和处理异常。

## 给JNI函数传递错误的参数
JNI函数不会检测参数是否合法，一个错误的参数可能导致未知的结果或者程序崩溃。JDK 1.2以上版本提供`-Xcheck:jni`命令行选项，让JVM去发现一些错误参数的场景。不管怎样，要确保传递给JNI函数的参数是合法的。

## 混淆`jclass`和`jobejct`
第一次使用JNI的时候可能会搞不清楚对象引用(`jobject`类型的值)和类引用(`jclass`类型的值)的区别。对象对应的是数组，java.lang.object或者它的子类的实例，而类引用对应的是java.lang.Class的实例。<br>
像`GetFieldID`这种需要一个`jclass`类型参数的方法是类操作。相反，`GetIntField`这种需要一个`jobject`类型参数的方法是实例操作。

## `jboolean`的数据截断问题
`jboolean`是一个8位unsigned的C类型，0对应的是`JNI_FALSE`而其余的值对应`JNI_TRUE`，但在位数大于8位的数据类型，就出现问题了(低8位全为0的非零值)。看看下面的例子:
```c
void print(jboolean condition) 
{ 
    /* C compilers generate code that truncates condition to its lower 8 bits. */    
    if (condition) {        
        printf("true\n");    
    } else {        
        printf("false\n");    
    } 
}
```
考虑这种情况:
```c
int n = 256; /* the value 0x100, whose lower 8 bits are all 0 */ 
print(n);
```
上面的结果偏离的预期，输出false。一个常用的解决办法:
```c
n = 256; 
print (n ? JNI_TRUE : JNI_FALSE);
```


## Java代码和C/C++代码的界限
关于界限问题这里有一些准则:
* 尽量简化Java和C/C++之间的接口，Java和C/C++之间的调用很复杂的话会增加调式和维护的难度。这种频繁的跨语言间的调用也会影响JVM的性能。比如，Java内联一个Java定义的方法要比内联一个C/C++定义的方法更高效。
* 最小化本地代码，本地代码是不可移植和类型不安全的，而且本地代码的异常检测也很繁琐。
* 尽量独立本地代码。实际使用中，尽量在一个包或者类中声明本地方法，和应用的其他部分隔绝。<br>

JNI提供了访问JVM的能力，比如类加载，对象操作，访问字段，回调函数，线程同步等等。虽然本地代码实现复杂的Java交互很诱人，但是Java的实现往往很简单。下面的例子说明为什么在本地代码写Java很蠢。想象一下在Java中创建线程:
```java
new JobThread().start(); 
```
在本地代码中实现相同的操作：
```c
/* Assume these variables are precomputed and cached: 
*     Class_JobThread:  the class "JobThread" 
*     MID_Thread_init:  method ID of constructor 
*     MID_Thread_start: method ID of Thread.start() 
*/ 
aThreadObject = (*env)->NewObject(env, Class_JobThread, MID_Thread_init); 
if (aThreadObject == NULL) {    
    ... /* out of memory */ 
} 
(*env)->CallVoidMethod(env, aThreadObject, MID_Thread_start); 
if ((*env)->ExceptionOccurred(env)) {    
    ... 
    /* thread did not start */ 
}
```
比较起来，本地代码会有很多异常检测，而且更容易出错。<br>
为了避免在本地代码中实现太过复杂的逻辑，我们会更倾向于使用回调方法在Java中实现。


## 混淆ID和引用
JNI通过引用访问对象(Classes, strings, arrays等)，通过ID来访问方法和字段。<br>
引用指向的是JVM的资源，可以由本地代码管理(比如`DeleteLocalRef`)；字段和方法ID由JVM控制，只有对应的类被unload的时候才会失效，本地方法不能显式删除这些ID。<br>
本地方法可以定义多个引用指向用一个同一个对象，比如可以有一个全局引用和局部引用指向同一对象的情况；而字段和方法ID则是唯一的。假设类B从类A中继承了方法`f`，下面的结果是一样的:
```c
jmethodID MID_A_f = (*env)->GetMethodID(env, A, "f", "()V");
jmethodID MID_B_f = (*env)->GetMethodID(env, B, "f", "()V");
```

## 缓存字段和方法ID
之前我们提到过为了效率，我们会缓存字段和方法ID，但有时候也不仅仅是为了性能。ID的缓存可以保证字段和方法能被本地方法正确的访问。下面看一个例子:
```java 
class C {    
    private int i;    
    native void f(); 
}
```
假设方法`f`需要访问字段`i`，下面是没有使用缓存的实现:
```c
// No field IDs cached. 
JNIEXPORT void JNICALL 
Java_C_f(JNIEnv *env, jobject this) 
{    
    jclass cls = (*env)->GetObjectClass(env, this);    
    ... /* error checking */    
    jfieldID fid = (*env)->GetFieldID(env, cls, "i", "I");
    ... /* error checking */    
    ival = (*env)->GetIntField(env, this, fid);    
    ... /* ival now has the value of this.i */ 
}
```
目前代码看着没什么bug，但是如果我们有一个C的子类D:
```java
// Trouble in the absence of ID caching 
class D extends C {    
    private int i;    
    D() {        
        f(); // inherited from C    
    } 
}
```
当D的构造函数调用`C.f`，此时本地方法获取的`this`指针指向D，也就是说`cls`和D相关，`fid`变成了`D.i`，这和预计的结果不一样。解决办法就是在合适的地方缓存字段ID：
```java
// Version that caches IDs in static initializers 
class C {    
    private int i;    
    native void f();    
    private static native void initIDs();    
    static {        
        initIDs(); // Call an initializing native method
    } 
}
```
本地代码实现:
```c
static jfieldID FID_C_i;

JNIEXPORT void JNICALL 
Java_C_initIDs(JNIEnv *env, jclass cls) 
{    
    /* Get IDs to all fields/methods of C that native methods will need. */    
    FID_C_i = (*env)->GetFieldID(env, cls, "i", "I"); 
}
JNIEXPORT void JNICALL 
Java_C_f(JNIEnv *env, jobject this) 
{    
    ival = (*env)->GetIntField(env, this, FID_C_i);    
    ... /* ival is always C.i, not D.i */ 
}
```
同样，缓存在方法调用上也适用。但缓存在虚拟函数调用是不需要的，因为虚拟函数是动态绑定的，这样你就可以使用工具函数`JNU_CallMethodByName`来调用虚拟函数。

## Unicode字符串的结尾符
从`GetStringChars`和`GetStringCritical`获取的Unicode字符串不是以NULL结尾的，需要调用`GetStringLength`获取Unicode字符的个数。在有些操作系统中，比如Windows，Unicode字符串需要两个`'\0'`结尾，所以不能直接将上述两个方法的结果赋值给Windows的Unicode字符串，需要手动添加两个`'\0'`。


## 破坏访问规则
本地代码访问字段和方法不受Java规则限制，比如可以访问和修改`private`和`final`。JNI的设计使得本地代码可以访问任意位置的heap内存，这会导致意想不到的结果。比如在一个JIT编译器内联了一个`final`字段和本地函数又对它进行修改后，就可能产生不一致性。另外，修改java.lang.String这种对象会破坏Java规范。

## 忽视国际化代码
这里主要是[字符串的编码格式](/blog/2019-10-11-JNI-guides-and-specifications-8#国际化代码)的问题，通常需要使用`JNU-NewStringNative`和`JNU_GetStringNativeCHars`等方法转化符合规则的字符串。
## 释放VM资源
常见错误就是在本地方法中，出现异常的时候忘记[释放JVM的资源](/blog/2019-10-09-JNI-guides-and-specifications-6#处理异常)。比如在调用`GetStringChar`时忘记调用`ReleaseStringChars`会导致`jstring`在JVM中被pin住。造成内存碎片化，或者C内存泄漏。

## 大量创建局部引用
[大量创建局部引用](/blog/2019-10-02-JNI-guides-and-specifications-5#释放局部引用)会造成不必要的内存浪费。注意管理好会长期执行的本地方法，循环以及工具函数中的的局部引用。利用好`Push/PopLocalFrame`来管理局部引用。

## 使用失效的局部引用
局部引用只在创建方法内有效(单线程)，不要使用全局变量储存或者传递给其他线程。

## 跨线程使用`JNIEnv`
[`JNIEnv`指针](/blog/2019-10-11-JNI-guides-and-specifications-8#在任意地方获取JNIEnv指针)只在单线程中有效，不能缓存该指针或者从其他线程获取。

## 线程模型不匹配
JNI涉及到线程的使用时，JVM和主机环境必须支持同一线程模型([线程模型的匹配](/blog/2019-10-11-JNI-guides-and-specifications-8#匹配线程模型))，否则，本地线程将不能附着到JVM中去。