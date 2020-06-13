---
title: JNI程序规范和指南3——基本类型, 字符串和数组
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---
一个开发者在Java和本地代码交互的过程中最常见的问题是Java的数据类型是如何映射到C/C++中去的。在上一章的例子中我们没有给本地方法传递参数，本地方法也没有返回结果。但是实际中，大多数程序是需要传递参数给本地方法以及从本地方法中获取返回值的。本文将介绍如何在Java和C/C++之间交换参数。<br>

本文会从Java的基本类型(比如整型), 以及常见的对象(比如字符串和数组)开始介绍，至于如何传递任意对象，访问字段和调用方法将在[下一篇](/blog/2019-09-30-JNI-guides-and-specifications-4)文章中介绍。

## 一个简单的例子

Java源代码：
```java
import java.lang.*;
class Prompt{
    public native String getLine(String s);
    public static void main(String args[]) {
        Prompt p=new Prompt();
        String input = p.getLine("Type a line:");
        System.out.println(input);
    }
    static{
        System.loadLibrary("Prompt");
    }
}
``` 
### 本地方法的C原型函数
在生成的头文件中可以看到Prompt.getLine的原型：
```c
JNIEXPORT jstring JNICALL Java_Prompt_getLine
  (JNIEnv *, jobject, jstring);
```
### 本地方法的形参
`Java_Prompt_getLine`中除了两个标准的形参`JNIEnv*`和`jobject`，还有一个从Java中传进来的`jstring`。JNIEnv指针指向了一个包含许多JNI方法的函数表, 如下图所示：
![the JNIEnv interface pointer]({{site.data.strings.blog_url}}JNI3-1.png)<br>
第二个参数根据本地方法是静态方法还是实例方法而不同：本地方法是静态方法时，第二个参数对应本地方法所在的类；当本地方法是实例方法时，第二个参数对应本地方法所在的对象。

### 类型映射
JNI中定义了一个映射于Java类型的C/C++类型集合。Java中有两种类型：一种是基本类型, 比如`int`, `float`, `char`; 一种是引用类型，比如类，对象和数组。<br>
JNI对基本类型和引用类型的处理是不同的。基本类型是直接映射的关系，比如Java的`int`对应C/C++中的`jint`, `float`对应`jfloat`(在jni.h中定义)。<br>
JNI把Java的对象当作是C指针类型传递到本地方法(native method)中，指向Java虚拟机中的一个内部数据结构，而内部数据结构在内存中的存储方式是不可见的。本地方法必须通过`JNIEnv*`函数表中的函数来处理这些内部数据结构。比如，本地方法需要通过`GetStringUTFChars`来获取Java中String的值。<br>
所有的JNI引用都是`jobject`类型，为了使用方便和类型安全，JNI定义了一个引用类型的集合，集合中所有的子类型(subtypes)都是`jobject`类型。这些子类型对应着Java中常用的引用类型，比如`jstring`对应字符串，`jobjectArray`对应数组对象。

## 访问字符串
`Java_Prompt_getLine`从`Prompt.getLine`获取了一个`jstring`类型的参数，`jstring`代表着Java虚拟机中的`String`而不是C/C++中的`char *`，也就是说你不能像使用C字符串那样使用`jstring`。比如下面的代码是有问题的:
```c
JNIEXPORT jstring JNICALL Java_Prompt_getLine
  (JNIEnv *env, jobject obj, jstring prompt){
      printf("%s",prompt);
  }
```
### 转换为本地的字符串
JNI支持Unicode和UTF-8字符串之间的转换。Unicode字符串将字符表示为16位值，而UTF-8字符串使用向上兼容7位ASCII字符串的编码方案。即使UTF-8字符串包含非ASCII字符，它也类似于以NULL结尾的C字符串。所有值在1到127之间的7位ASCII字符在UTF-8编码中保持不变。一个字节如果最高位被设置了，意味着这是一个多字节字符(16-bit Unicode值)。<br>
`Java_Prompt_getLine`通过调用JNI函数`GetStringUTFChars`读取字符串的内容。`GetStringUTFChars`是`JINEnv*`指向的函数表中的一个函数，可以将Java虚拟机中的字符串对象引用(Unicode序列)转换为C/C++中的UTF-8格式的字符串。如果你确定Java字符串只包含7位的ASCII字符，则可以将转换后的字符串传递给C库函数使用(比如`printf`)。
```c
#include <jni.h>
#include <stdio.h>
#include "Prompt.h"

JNIEXPORT jstring JNICALL Java_Prompt_getLine
    (JNIEnv *env, jobject obj, jstring prompt) {    
    char buf[128];    
    const jbyte *str;    
    str = (*env)->GetStringUTFChars(env, prompt, NULL);    
    if (str == NULL) {        
        return NULL; 
        /* OutOfMemoryError already thrown */    
    }    
    printf("%s", str);    
    (*env)->ReleaseStringUTFChars(env, prompt, str);    
    /* We assume here that the user does not type more than 127 characters */    
    scanf("%s", buf);    
    return (*env)->NewStringUTF(env, buf); 
}
```
不要忘记检查`GetStringUTFChars`的返回值，因为JVM需要为新创建的UTF-8字符串分配内存，这里可能会因为内存不足而创建失败。一旦创建失败，`GetStringUTFChars`会返回一个`NULL`并抛出一个`OutOfMemoryError`异常。JNI抛出一个异常和Java中的异常是不一样的，这会在第六章做说明。一个由JNI抛出的异常是不会改变C/C++的执行的，所以需要显式的调用`return`结束程序。`Java_Prompt_getLine`返回后，`Prompt.main`(`Prompt.getLine`的调用者)会抛出一个异常。

### 释放本地字符串资源
当本地方法不再使用`GetStringUTFChars`中获取的字符串后，需要调用`ReleaseStringUTFChars`释放字符串的资源，否则会造成内存泄露。
### 构造新的字符串
你可以调用`NewStringUTF`在本地方法中构造一个`java.lang.String`对象(UTF-8 --> Unicode)。同样的，`NewStringUTF`在无法获取足够内存的时候会返回`null`并抛出`OutOfMemoryError`异常。在上面的例子中我们并没有检查是否创建成功，是因为本地方法立即返回了(一般不会出问题)。如果创建成功, `Prompt.getLine`会返回一个`String`类型的对象。
### 其他JNI字符串函数
除了`GetStringChars`, `ReleaseStringChars`, `NewStringUTF`之外，JNI还支持其他的字符串处理函数。<br>
`GetStringChars`和`ReleaseStringChars`获取一个Unicode类型的字符串，这两个函数在支持Unicode编码的字符串的操作系统中很有用。UTF-8格式的字符串总是以`'\0'`结尾，而Unicode字符串则不是。你可以使用`GetStringLength`获取`jstring`引用中的Unicode字符的个数。如果想知道在UTF-8格式中需要多少字节来表示一个`jstring`，可以调用ANSI C的`strlen`来获取`GetStringUTFChars`返回值的长度，也可以直接使用`GetStringLength`获取`jstring`的长度。<br>
`GetStringCHars`和`GetStringUTFChars`的第三个参数需要进一步说明：
```
const jchar * 
GetStringChars(JNIEnv *env, jstring str, jboolean *isCopy);
```
JNI函数`GetStringChars`返回值是原字符串(`java.lang.String`)的拷贝时，`isCopy`被赋值为`JNI_TRUE`; 如果返回值是直接指向原字符串的指针, 则`isCopy`被赋值为`JNI_FALSE`, 此时本地方法绝不能修改该字符串的值，否则JVM中的字符串值也会跟着改变，这将违反了Java中字符串不可更改的规则。<br>
一般情况下你不关心JVM的返回值是否为拷贝，你需要传一个`NULL`进去。<br>
虚拟机是否会返回一个拷贝是不可预测的，最好假设这类函数返回的是一个拷贝，而这会花费一定的时间和空间。一个典型的JVM，其垃圾回收器(GC)会为heap上的对象重新分配内存，一旦`GetStringChars`这类JNI函数直接返回JVM中该对象的指针，则GC不再为这个对象重新分配内存，也就是说JVM必须`pin`住这个对象，大量的`pin`就会导致内存碎片化。<br>
当你不再使用这个字符串对象，你需要调用`ReleaseStringChars`。字符串是释放还是`unpin`取决于`GerStringChars`是返回一个拷贝还是对象的指针。

### JDK 1.2中的JNI函数
为了提高Java虚拟机直接返回字符串(java.lang.String)指针的可能性，JDK 1.2发布了一对新函数`Get/ReleaseStringCritical`。表面上和`Get/ReleaseStringChars`很相似，都是尽可能的返回字符串的指针，但是这对函数在使用上有很大的限制。<br>
你需要确保这对函数之间的代码是运行在**critical region**(临界区域)，也就是说在临界区域本地代码不能调用任意会导致当前线程阻塞或者等待JVM中其他线程的JNI函数或者本地函数(除了`Get/ReleaseStringCritical`和`Get/ReleasePrimitiveArrayCritical`)。比如当前线程就不能够等待另一读取输入的线程。<br>
这些限制使得Java虚拟机可以在从`GetStringCritical`中直接获取一个字符指针时禁用GC。当禁用GC时，其他触发GC的线程会被阻塞。在`Get/ReleaseStringCritical`之间的本地代码不能触发阻塞调用或者在JVM中给一个对象分配内存，否则Java虚拟机会死锁：
* 在当前线程完成了阻塞性调用并重新启用(reenable)GC之前，其他线程触发的GC不能执行直到。
* 同时，当前线程不会执行：因为阻塞性调用需要获取其他线程正在持有的锁，而持有锁的线程在等待GC。

`Get/ReleaseStringCritical`的交叠调用是安全的:
```c
jchar *s1, *s2; 
s1 = (*env)->GetStringCritical(env, jstr1); 
if (s1 == NULL) {    
  ... /* error handling */ 
} 
s2 = (*env)->GetStringCritical(env, jstr2); 
if (s2 == NULL) {    
  (*env)->ReleaseStringCritical(env, jstr1, s1);    
  ... /* error handling */ 
} 
...     /* use s1 and s2 */ 
(*env)->ReleaseStringCritical(env, jstr1, s1); 
(*env)->ReleaseStringCritical(env, jstr2, s2);
```
`GetStringCritical`和`ReleaseStringCritical`的嵌套不需要严格遵守堆栈顺序。但是我们还得判空，因为当JVM以不同的格式存储内部数据结构时，`GetStringCritical`还是有可能会返回一个字符串的拷贝。比如说JVM中存储的数组(*这里应该表示的是字符串序列*)是不连续的，这种情况下拷贝一份连续存储的再返回给本地代码。<br>
另外的新函数是`GetStringRegion`和`GetStringUTFRegion`。这些函数会复制字符串到一个预先分配好的缓存中。`Prompt.getLine`的另一个实现版本：
```c
JNIEXPORT jstring JNICALL 
Java_Prompt_getLine(JNIEnv *env, jobject obj, jstring prompt) 
{ 
  /* assume the prompt string and user input has less than 128 characters */    
  char outbuf[128], inbuf[128];    
  int len = (*env)->GetStringLength(env, prompt);    
  (*env)->GetStringUTFRegion(env, prompt, 0, len, outbuf);  printf("%s", outbuf);    
  scanf("%s", inbuf);    
  return (*env)->NewStringUTF(env, inbuf); 
}
```
`GetStringUTFRegion`需要一个开始索引和长度(以Unicode字符的数量进行计算), 同时还有边界检查，会抛出`StringIndexOutOfBoundsExcption`异常(*上面代码没有判断输入字符长度是否小于128*)。<br>
> `GetStringUTFChars`是不会产生内存分配的(no memory allocation)，所以不需要判空。

### JNI字符串函数总结
下面的表格总结了字符串相关的JNI函数。<br>

| JNI fonuction | Description | Since |
|:--------------|:------------|:------|
| `GetStringUTFChars`, `ReleaseStringUTFChars` | 获取或者释放Unicode格式的字符串, 可能会返回一个拷贝 | JDK 1.1|
| `GetStringUTFChars`, `ReleaseStringUTFChars`| 获取或者释放一个UTF-8格式的字符串, 可能会返回一个拷贝 | JDK 1.1|
| `GetStringLength` | 返回Unicode字符的个数 | JDK 1.1|
| `GetStringUTFLength` | 返回表示一个UTF-8字符串所需要的字节数, 不包括`'\0'`| JDK 1.1|
|`NewString` | 创建一个java.lang.String字符串对象(Unicode)|JDK 1.1|
| `NewStringUTF` | 创建一个java.lang.String字符串对象(UTF-8) | JDK 1.1|
| `GetStringCritical`, `ReleaseStringCritical`| 获取一个指向Unicode字符串的指针，可能会返回一个拷贝，no blocking | JDK 1.2|
|`GetStringRegion`, `ReleaseStringRegion`|复制Unicode字符串到预先分配的C缓冲区| JDK 1.2|
| `GetStringUTFRegion`, `ReleaseStringUTFRegion`| 复制UTF-8字符串到预先分配的C缓冲区| JDK 1.2|

### 如何选择字符串函数
![Choosing among the JNI String Functions]({{site.data.strings.blog_url}}JNI3-2.png)<br>

使用`GetStringCritical`的时候要特别小心。比如以下代码就有可能会造成死锁：
```c
/* This is not safe! */ 
const char *c_str = (*env)->GetStringCritical(env, j_str, 0); 
if (c_str == NULL) {    
  ... /* error handling */ 
} 
fprintf(fd, "%s\n", c_str); 
(*env)->ReleaseStringCritical(env, j_str, c_str);
```
上面的代码是不安全的，假设有另外一个线程T正在等待读取`fd`，而此时OS的缓存规则是`fpringf`需要等待线程T读取`fd`完成后才能执行，如果此时线程T没有足够的内存来读取文件，则需要调用GC，而GC已被`GetStringCritical`禁用，最后死锁。<br>

## 访问数组
JNI处理基本类型数组(primitive arrays)和对象数组(object arrays)的方式是不一样的。
```c
//primitive arrays
int[] iaee;
float[] farr;
//object arrays
Object[] oarr;
int[][] att2;
```
下面是一个简单的例子, 调用`sumArray`将数组元素相加：
```java
import java.lang.*;
class IntArray {    
    private native int sumArray(int[] arr);    
    public static void main(String[] args) {        
        IntArray p = new IntArray();        
        int arr[] = new int[10];        
        for (int i = 0; i < 10; i++) {            
            arr[i] = i;        
        }        
        int sum = p.sumArray(arr);        
        System.out.println("sum = " + sum);    
    }    
    static {        
        System.loadLibrary("IntArray");    
    } 
}
```
### C中访问数组
数组被表示为`jarray`，但`jarray`并不是C/C++中的数组类型，以下的代码是错误的:
```c
/* This program is illegal! */ 
JNIEXPORT jint JNICALL 
  Java_IntArray_sumArray(JNIEnv *env, jobject obj, jintArray arr){    
    int i, sum = 0;    
    for (i = 0; i < 10; i++) 
    {        
      sum += arr[i];    
    } 
  }
```
必须使用合适的JNI函数来访问数组元素:
```c
JNIEXPORT jint JNICALL 
Java_IntArray_sumArray(JNIEnv *env, jobject obj, jintArray arr){    
  jint buf[10];    
  jint i, sum = 0;    
  (*env)->GetIntArrayRegion(env, arr, 0, 10, buf);    
  for (i = 0; i < 10; i++) 
  {        
    sum += buf[i];    
  }    
  return sum; 
}
```

### 访问基本类型数组
上面的例子中，`GetIntArrayRegion`函数复制整型数组的全部元素到C缓冲区`buf`中，第三个参数是元素的开始索引，第四个参数是要复制的元素个数。这里会有内存溢出问题。<br>
JNI支持使用`SetIntArrayRegion`来对整型数组进行修改。其他基本类型的数组同样支持。<br>
JNI支持`Get/Release<Type>ArrayElements`函数集合来获取基本类型数组的元素。由于GC不一定支持`pin`操作，所以Java虚拟机一般会返回一个指向拷贝数据缓冲区的指针。之前的代码可以修改为:
```c
JNIEXPORT jint JNICALL 
Java_IntArray_sumArray(JNIEnv *env, jobject obj, jintArray arr) {    
  jint *carr;    
  jint i, sum = 0;    
  carr = (*env)->GetIntArrayElements(env, arr, NULL);    
  if (carr == NULL) {        
    return 0; 
    /* exception occurred */    
  }    
  for (i=0; i<10; i++) {        
    sum += carr[i];    
  }    
  (*env)->ReleaseIntArrayElements(env, arr, carr, 0);    return sum; 
}
```
`GetArrayLength`函数返回数组元素的个数，这个值在数组创建的时候就被确定下来了。<br>
和字符串类似，JDK 1.2也支持`Get/ReleasePrimitiveArrayCritical`函数。
### JNI基本类型数组函数总结

| JNI Function | Description | Since |
|:-------------|:------------|:------|
| `Get<Type>ArrayRegion`, `Set<Type>ArrayRegion` | 复制数组内容到C buffer或者从C buffer复制数组内容 | JDK 1.1 |
| `Get<Type>ArrayElements`, `Release<Type>ArrayElements` | 获取或释放指向数组的指针，可能会返回一个拷贝 | JDK 1.1|
| `GetArrayLength` | 返回数组元素个数 | JDK 1.1 |
| `New<Type>Array` | 创建给定长度的数组  | JDK 1.1|
| `GetPrimitiveArrayCritical`, `ReleasePrimitieArrayCritical` | 获取或者释放数组，禁用GC，可能会返回数组的拷贝 | JDK 1.2|

### 如何选择基本类型数组函数
![Choosing among Primitive Array Functions]({{site.data.strings.blog_url}}JNI3-3.png) <br>

## 访问对象数组
JNI提供了单独的函数对来访问对象数组。`GetObjectArrayElement`返回一个给定索引的对象数组元素，`SetObjectArrayElement`更新给定索引的数组元素。和基本类型数组不同的是，你不能一次性的获取或拷贝对象数组中的全部对象。字符串和数组都是引用类型，你可以使用`Get/SetObjectArrayElement`来访问字符串数组或多维数组(n>2)。<br>
下面的例子调用一个本地方法创建一个二维整型数组并打印数组的内容:
```java 
import java.lang.*;

class ObjectArrayTest {    
    private static native int[][] initInt2DArray(int size);    
    public static void main(String[] args) {        
        int[][] i2arr = initInt2DArray(3);        
        for (int i = 0; i < 3; i++) {            
            for (int j = 0; j < 3; j++) {                 
                System.out.print(" " + i2arr[i][j]);            
            }            
            System.out.println();        
        }    
    }    
    static {       
        System.loadLibrary("ObjectArrayTest");    
    } 
}
```
本地方法`initInt2DArray`创建了一个给定大小的二维数组：
```c
#include <jni.h>
#include <stdio.h>
#include "ObjectArrayTest.h"

JNIEXPORT jobjectArray 
JNICALL Java_ObjectArrayTest_initInt2DArray(JNIEnv *env, jclass cls, int size) {    
    jobjectArray result;    
    int i;    
    jclass intArrCls = (*env)->FindClass(env, "[I");    
    if (intArrCls == NULL) {        
        return NULL; 
        /* exception thrown */    
    }    
    result = (*env)->NewObjectArray(env, size, intArrCls, NULL);    
    if (result == NULL) {        
        return NULL; 
        /* out of memory error thrown */    
    }    
    for (i = 0; i < size; i++) {        
        jint tmp[256];  
        /* make sure it is large enough! */        
        int j;        
        jintArray iarr = (*env)->NewIntArray(env, size);        
        if (iarr == NULL) {            
            return NULL; 
            /* out of memory error thrown */        
        }        
        for (j = 0; j < size; j++) {            
            tmp[j] = i + j;        
        }        
        (*env)->SetIntArrayRegion(env, iarr, 0, size, tmp);        
        (*env)->SetObjectArrayElement(env, result, i, iarr);        
        (*env)->DeleteLocalRef(env, iarr);    
    }    
    return result; 
}
```
`Java_ObjectArrayTest_initInt2DArray`方法首先调用JNI函数`FindClass`获取一个二维整型数组的元素的类的引用，传递给`FindClass`的`[I`是**JNI类型描述符**(JNI class descriptor)，表示的是Java虚拟机中的`int[]`类型。如果类型加载失败则返回`null`并抛出一个异常。<br>
`NewObjectArray`创建一个数组，其元素的类型为`intArrayCls`类型引用定义的类型。`NewObjectArray`函数只能分配第一维，第二维相当于使用一个一维数组作为元素类型填充。Java虚拟机没有专门的数据结构来描述多维数组，一个二维数组就是一个数组的数组。<br>
创建第二维数组的方法很直接，`NewIntArray`为每一个数组分配空间，`SetIntArrayRegion`复制缓冲区`tmp[]`的内容到刚刚新创建的一维数组，在完成`SetObjectArrayElement`的调用后，第i个一维数组的第j个元素的值是`i+j`。<br>
在循环的末尾调用`DeleteLocalRef`释放临时资源。


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