---
title: JNI程序规范和指南6——异常
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---

在本地代码中调用JNI函数时，我们已经处理过很多可能存在错误的情况。本文将介绍本地代码如何发现并处理这些错误。<br>

本文着重介绍的是调用JNI函数时出现的错误，至于其他代码的错误，比如调用系统方法发生的错误，只需要按照系统文档来处理就行。当调用JNI函数时，需要按照本文介绍的步骤来检查和处理可能存在异常。

## 概述
通过一系列的例子来介绍JNI异常处理函数。

### 本地方法缓存和抛出异常
下面的例子说明了如何声明一个抛出异常的本地方法。`CatchThrow`类声明了一个抛出`IllegalArgumentException`的`doit`函数：
```java
import java.lang.*;

class CathcThrow{
    private native void doit()        
        throws IllegalArgumentException;    
    private void callback() throws NullPointerException {        
        throw new NullPointerException("CatchThrow.callback");    
    } 
    public static void main(String args[]) {        
        CatchThrow c = new CatchThrow();        
        try {            
            c.doit();        
        } catch (Exception e) {            
            System.out.println("In Java:\n\t" + e);        
        }    
    }    
    static {        
        System.loadLibrary("CatchThrow");    
    }  
}
```
`CatchThrow.main`函数调用的本地方法定义如下:
```c
#include <stdio.h>
#include <jni.h>
#include "CatchThrow.h"

JNIEXPORT void JNICALL 
Java_CatchThrow_doit(JNIEnv *env, jobject obj) {    
    jthrowable exc;    
    jclass cls = (*env)->GetObjectClass(env, obj);    
    jmethodID mid = (*env)->GetMethodID(env, cls, "callback", "()V");    
    if (mid == NULL) {        
        return;    
    }    
    (*env)->CallVoidMethod(env, obj, mid);    
    exc = (*env)->ExceptionOccurred(env);    
    if (exc) {        
        /* We don't do much with the exception, except that we print a debug message for it, 
        clear it, and  throw a new exception. */        
        jclass newExcCls;        
        (*env)->ExceptionDescribe(env);        
        (*env)->ExceptionClear(env);        
        newExcCls = (*env)->FindClass(env, "java/lang/IllegalArgumentException");        
        if (newExcCls == NULL) {            
            /* Unable to find the exception class, give up. */            
            return;        
        } 
        (*env)->ThrowNew(env, newExcCls, "thrown from C code");    
    } 
}
```
执行结果如下:
```
Exception in thread "main" java.lang.NullPointerException: CatchThrow.callback
        at CatchThrow.callback(CatchThrow.java:7)
        at CatchThrow.doit(Native Method)
        at CatchThrow.main(CatchThrow.java:12)
In Java:
        java.lang.IllegalArgumentException: thrown from C code
```
`callback`函数抛出了`NullPointerException`，当`CallVoidMethod`将控制权返回给本地代码后，本地代码通过`ExceptionOccured`检测异常。在例子中，检测到异常则通过`ExceptionDescribe`输出异常信息，通过`ExceptionClear`清除异常，然后再抛出一个`IllegalArgumentException`。<br>

通过JNI引发的未决异常(例如，通过调用`ThrowNew`)不会立即中断本机方法的执行。这和Java是不一样的。Java中一旦抛出异常，就会将程序的控制权交给最近的能够匹配异常类型的`try/catch`，然后虚拟机会清除异常然后执行异常处理。异常发生之后，开发者必须手动处理异常。<br>

### 一个工具函数
抛出异常首先得发现异常类然后调用`ThrowNew`，为了简化操作，可以实现一个工具函数:
```c
void 
JNU_ThrowByName(JNIEnv *env, const char *name, const char *msg) 
{    
    jclass cls = (*env)->FindClass(env, name);    
    /* if cls is NULL, an exception has already been thrown */    
    if (cls != NULL) {        
        (*env)->ThrowNew(env, cls, msg);    
    }    
    /* free the local ref */    
    (*env)->DeleteLocalRef(env, cls); 
}
```
这系列文章中，只要带有`JNU`前缀的函数都是工具函数。`JNU_ThrowByName`首先通过`FindClass`查找异常类，如果没有找到，虚拟机会抛出一个`NoClassDefFoundError`的异常。如果找到了，就会抛出对应的异常，`JNU_THrowByName`返回时，会保证有一个未决的异常，但这个异常不一定就是name参数指定的异常。最后我们必须确保局部引用被释放。<br>

## 异常处理
JNI程序员需要提前知道可能的异常发生条件并检测和处理这些异常。妥善的处理异常是程序稳定的必要条件。

### 检测异常
这里有两种方法来检测异常:
1. 大多数的JNI函数通过返回一个特定值(比如NULL)来表示发生了一个错误，同时也意味着当前线程有一个未决异常(在c中，使用返回值来表示异常是很常见的)。在下面的例子如何使用返回值NULL来检测错误。例子包含两个部分，一部分是`Window`类定义了一些字段和一个本地方法缓存了字段的ID，即使`Window`类定义了这些字段，我们仍然需要检测`GetFieldID`返回的NULL，因为虚拟机可能没有足够的内存来存储字段ID。
```java
/* a class in the Java programming language */ 
public class Window {    
    long handle;    
    int length;    
    int width;    
    static native void initIDs();    
    static {        
        initIDs();    
    } 
}
```
```c
/* C code that implements Window.initIDs */ 
jfieldID FID_Window_handle; 
jfieldID FID_Window_length; 
jfieldID FID_Window_width;
JNIEXPORT void JNICALL 
Java_Window_initIDs(JNIEnv *env, jclass classWindow) 
{    
    FID_Window_handle = (*env)->GetFieldID(env, classWindow, "handle", "J");    
    if (FID_Window_handle == NULL) {  
        /* important check. */        
        return; 
        /* error occurred. */    
    }    
    FID_Window_length = (*env)->GetFieldID(env, classWindow, "length", "I");    
    if (FID_Window_length == NULL) {  
        /* important check. */        
        return; /* error occurred. */    
    }    
    FID_Window_width = (*env)->GetFieldID(env, classWindow, "width", "I");    
    /* no checks necessary; we are about to return anyway *
}
```

2. 如果一个JNI函数的返回值不能标记一个错误，那么就需要使用`ExceptionOccurred`来检测未决异常(JDK 1.2中添加了`ExceptionCheck`)。比如`CallIntMethod`的返回值无法标记一个错误，常见的返回值NULL和-1不能用作错误标记。看下面的例子:
```java
public class Fraction {    
    // details such as constructors omitted    
    int over, under;    
    public int floor() {        
        return Math.floor((double)over/under);    
    } 
}
```
```c
/* Native code that calls Fraction.floor. Assume method ID  
MID_Fraction_floor has been initialized elsewhere. */ 
void f(JNIEnv *env, jobject fraction) 
{    
    jint floor = (*env)->CallIntMethod(env, fraction,  MID_Fraction_floor);    
    /* important: check if an exception was raised */    
    if ((*env)->ExceptionCheck(env)) {        
        return;    
    }    
    ... 
    /* use floor */ 
}
```
当JNI函数可以返回一个错误码，也可以使用显式方法来检测错误的发生，但是使用返回值检测会比较高效。**一旦JNI函数的返回值是一个错误码，那么接下来调用`ExceptionCheck`肯定会返回JNI_TRUE**。


### 处理异常
本地代码有两种方式处理未决异常:
* 本地方法可以选择立即返回，将异常返回给调用者处理。
* 本地方法也可以使用`ExceptionClear`清除异常，然后由本地代码处理异常。<br>

**出现异常后，一定要检测，处理和清除异常后再调用其他的JNI函数**。在未清除异常时调用大多数的JNI函数可能会造成意想不到的结果，此时只有一少部分用于处理异常和清除虚拟机资源的函数可以被调用。<br>

在异常发生时，能够释放虚拟机资源是很重要的。以下例子中，通过`GetStringChars`获取一个字符串，在异常发生时，手动清除字符串的资源:
```c
JNIEXPORT void JNICALL 
Java_pkg_Cls_f(JNIEnv *env, jclass cls, jstring jstr) 
{    
    const jchar *cstr = (*env)->GetStringChars(env, jstr); 
    if (c_str == NULL) {        
        return;    
    }    
    ...    
    if (...) { /* exception occurred */        
        (*env)->ReleaseStringChars(env, jstr, cstr); 
        return;    
    }    
    ...    
    /* normal return */    
    (*env)->ReleaseStringChars(env, jstr, cstr); 
}
```

### 工具函数中的异常
在编写工具函数时要将异常传递给调用者。以下有两个需要注意的问题:
* 最理想的情况，工具函数能够返回一个错误码，这会减轻检查未决异常的花销。
* 异常发生时需要小心的处理局部引用。<br>

为了说明，以下例子是一个工具函数，该函数根据名字和描述符调用对应的回调函数。
```c
jvalue JNU_CallMethodByName(JNIEnv *env, 
    jboolean *hasException,                     
    jobject obj,                     
    const char *name,                     
    const char *descriptor, ...) 
{    
    va_list args;    
    jclass clazz;    
    jmethodID mid;
    jvalue result;    
    if ((*env)->EnsureLocalCapacity(env, 2) == JNI_OK) {
        clazz = (*env)->GetObjectClass(env, obj);
        mid = (*env)->GetMethodID(env, clazz, name, descriptor);        
        if (mid) {            
            const char *p = descriptor;            
            /* skip over argument types to find out the  return type */            
            while (*p != ')') p++;            
            /* skip ')' */            
            p++;            
            va_start(args, descriptor);            
            switch (*p) {            
                case 'V':                
                    (*env)->CallVoidMethodV(env, obj, mid, args);                
                    break;            
                case '[':            
                case 'L':                
                    result.l = (*env)->CallObjectMethodV( env, obj, mid, args);                
                    break;            
                case 'Z':                
                    result.z = (*env)->CallBooleanMethodV(env, obj, mid, args);
                    break;            
                case 'B':                
                    result.b = (*env)->CallByteMethodV(env, obj, mid, args);                
                    break;            
                case 'C':                
                    result.c = (*env)->CallCharMethodV(env, obj, mid, args);                
                    break;            
                case 'S':                
                    result.s = (*env)->CallShortMethodV(env, obj, mid, args);                
                    break;            
                case 'I':                
                    result.i = (*env)->CallIntMethodV(env, obj, mid, args);                
                    break;            
                case 'J':                
                    result.j = (*env)->CallLongMethodV(env, obj, mid, args);                
                    break;            
                case 'F':                
                    result.f = (*env)->CallFloatMethodV(env, obj, mid, args);                
                    break;            
                case 'D':                
                    result.d = (*env)->CallDoubleMethodV(env, obj, mid, args);
                    break;            
                default:                
                (*env)->FatalError(env, "illegaldescriptor");            
            }            
            va_end(args);        
        }        
        (*env)->DeleteLocalRef(env, clazz);    
    }    
    if (hasException) {        
        *hasException = (*env)->ExceptionCheck(env);    
    }    
    return result; 
}
```
`JNU_CallMethodByName`有一个`jboolean`参数，如果一切正常，则赋值`JNI_FALSE`，如果出现异常，则赋值`JNI_TRUE`，方便调用者检测异常。<br>
`JNU_CallMethodByName`首先确保能够创建两个局部引用：一个类引用，一个返回值。然后从类引用中获取对象，查询方法ID。根据返回类型调用相应的JNI调用函数。回调过程完成后，如果`hasException`不是NULL，我们调用`ExceptionCheck`检查异常。<br>
函数`ExceptionCheck`和`ExceptionOccurred`非常相似，不同的地方是，当有异常发生时，`ExceptionCheck`不会返回一个指向异常对象的引用，而是返回`JNI_TRUE`，没有异常时，返回`JNI_FALSE`。而`ExceptionCheck`这个函数不会返回一个指向异常对象的引用，它只简单地告诉本地代码是否有异常发生。上面的代码如果使用`ExceptionOccurred`的话，应该这么写：
```c
if (hasException) {        
    jthrowable exc = (*env)->ExceptionOccurred(env);
    *hasException = exc != NULL;        
    (*env)->DeleteLocalRef(env, exc);  
}
```
注意释放局部引用。<br>

使用`JNU_CallMethodByName`可以重写`InstanceMethodCall.nativeMetho`:
```c
JNIEXPORT void JNICALL 
Java_InstanceMethodCall_nativeMethod(JNIEnv *env, jobject obj) 
{    
    printf("In C\n");    
    JNU_CallMethodByName(env, NULL, obj, "callback", "()V");
}
```
调用`JNU_CallMethodByName`函数后，我们不需要检查异常，因为本地方法后面会立即返回。


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