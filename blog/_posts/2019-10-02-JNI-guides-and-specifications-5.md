---
title: JNI程序规范和指南5——JNI中的局部引用和全局引用
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---

JNI以不透明引用(opaque references)的方式提供了一些实例和数组类型(`jobject`, `jclass`, `jstring`, `jarray`等)以供本地方法使用。本地代码不能直接访问不透明引用所指向的内容，而只能借助JNI方法去访问这些数据结构，这样开发者就不用考虑JVM中的对象的存储方式。这样的话，必须要了解JNI中的几种引用:<br>

* JNI提供三种不透明引用: 局部引用，全局引用和弱全局引用。
* 局部引用和全局引用有着不同的生命周期。局部引用会自动被释放，而全局引用和弱全局引用则需要程序员手动释放，否则会一直存在。
* 局部引用和全局引用会阻止GC回收所指向的对象，而弱全局引用则允许垃圾回收。
* 不是所有的引用都能在整个程序的上下文中被使用，比如在函数返回后继续使用局部引用(相对于该函数而言)是不合法的。<br>

下面将会详细的介绍JNI的引用，合理使用引用是写出高质量代码的关键。
## 局部引用和全局引用
下面介绍局部引用和全局引用的区别。

### 局部引用
大多数JNI函数会创建局部引用。比如，`NewObject`创建一个新实例并返回它的局部引用。<br>
局部引用只在创建它的函数返回之前有效，一旦函数返回，局部引用就会被释放。你不能将局部引用缓存在一个静态变量中以供下一次函数调用时使用。以下就是一个**错误**的例子:
```c
/* This code is illegal */ 
jstring 
MyNewString(JNIEnv *env, jchar *chars, jint len) {    
    static jclass stringClass = NULL;    
    jmethodID cid;    
    jcharArray elemArr;    
    jstring result;
    if (stringClass == NULL) {        
        stringClass = (*env)->FindClass(env, "java/lang/String");        
        if (stringClass == NULL) {            
            return NULL; 
            /* exception thrown */        
        }    
    }    
    /* It is wrong to use the cached stringClass here, because it may be invalid. */    
    cid = (*env)->GetMethodID(env, stringClass, "<init>", "([C)V");    
    ...    
    elemArr = (*env)->NewCharArray(env, len);    
    ... 
    result = (*env)->NewObject(env, stringClass, cid, elemArr);    
    (*env)->DeleteLocalRef(env, elemArr);    return result; 
}
```

例子中使用`stringClass`缓存引用是为了减少重复调用`FindClass(env, "java/lang/String");`的消耗。但是问题在于`FindClass`返回的是一个局部引用。为了说明问题在哪，我们看看下面的代码:
```c
JNIEXPORT jstring JNICALL 
Java_C_f(JNIEnv *env, jobject this) 
{    
    char *c_str = ...;    
    ...    
    return MyNewString(c_str); 
}
```
在Java中C.f返回之后，JVM会释放所有的`JAVA_C_F`中创建的局部有引用，包括存储在`stringClass`中的局部引用。再次调用`MyNewString`的时候就会访问一个不合法的局部引用(*此时`stringClass`不为空*)。<br>
这里有两种使局部引用失效的方法，一个是前文提到的JVM在本地方法返回时自动释放局部引用，另一种是程序员通过JNI函数`DeleteLocalRef`显式的管理局部引用的生命周期。<br>
既然JVM会自动释放所有的局部引用，为什么还要手动的管理？局部引用会在失效之前阻止GC释放资源，JVM只能等到调用`MyNewString`的函数(比如，`C.f`)返回后才能释放资源，而`DeleteLocalRef`则可以立即释放。局部引用在销毁之前可能会被传递给多个本地方法。<br>
局部引用只在创建它的线程中有效。不能使用全局变量来存储局部引用然后将其传递给其他的线程。

### 全局引用
全局引用可以跨线程使用，在程序员显式的释放之前，它都是有效的。和局部引用一样，全局引用也会阻止GC释放资源。<br>
但和局部引用能被绝大多数JNI函数创建不一样的是，全局引用只能被`NewGlobalRef`创建。以下是一个创建的例子:
```c
jstring 
MyNewString(JNIEnv *env, jchar *chars, jint len) 
{    
    static jclass stringClass = NULL;    
    jmethodID cid;    
    jcharArray elemArr;    
    jstring result;
    if (stringClass == NULL) { 
        jclass localClass = NULL;    
        localClass = (*env)->FindClass(env, "java/lang/String");        
        if (localClass == NULL) {            
            return NULL; 
            /* exception thrown */        
        }    
        stringClass=(*env)->NewGlobalRef(env,localClass);
        (*env)->DeleteLocalRef(env,localClass);
    }    
     
    cid = (*env)->GetMethodID(env, stringClass, "<init>", "([C)V");    
   
    elemArr = (*env)->NewCharArray(env, len);    

    (*env)->SetCharArrayRegion(env, elemArr, 0, len, chars);
    result = (*env)->NewObject(env, stringClass, cid, elemArr);    
    (*env)->DeleteLocalRef(env, elemArr);    
    return result; 
}
```
调用`NewGlobalRef`之后，`MyNewString`可以被多次调用。

### 弱全局引用
弱全局引用是JDK1.2之后的特性，通过`NewGlobalWeakRef`和`DeleteGlobalWeakRef`创建和销毁。弱全局引用有着和全局引用一样的生命周期(跨调用和跨线程), 但是有可能会自动被GC释放。在`MyNewString`的例子中，可以有选择的使用全局引用或者弱全局引用来存储java.lang.String类，而且他们之间没有什么区别: java.lang.String是一个系统类不会被GC回收。<br>
当本地代码中缓存的引用不必阻止GC回收所指向的对象时，弱全局引用就是一个最好的选择。例如，本地方法`mypkg.MyCls.f`需要缓存一个`mypkg.MyCls2`引用，此时缓存的引用仍能够被GC回收。
```c
JNIEXPORT void JNICALL 
Java_mypkg_MyCls_f(JNIEnv *env, jobject self) {    
    static jclass myCls2 = NULL;    
    if (myCls2 == NULL) {        
        jclass myCls2Local = (*env)->FindClass(env, "mypkg/MyCls2");        
        if (myCls2Local == NULL) {            
            return; 
            /* can’t find class */        
        }       
        myCls2 = NewWeakGlobalRef(env, myCls2Local);        if (myCls2 == NULL) {            
            return; 
            /* out of memory */        
        }    
    } 
    ... /* use myCls2 */
}
```
假设`MyCLs`和`MyCls2`有相同的生命周期(例如，被同一个[类加载器]加载)。这样就不用考虑`MyCls`和它的本地方法在被使用时，`MyCls2`被重新加载的情况。如果上述情况仍存在，则需要检查弱引用所指向的对象是否已被GC回收。后面会介绍如何检查弱引用。

### 引用的对比
对于给定的两个引用(局部，全局，弱全局)，可以检查他们是否指向同一个对象:

```c
(*env)->IsSameObject(env, obj1, obj2)
```
当obj1和obj2指向同一个对象时返回`JNI_TRUE`。JNI中，一个NULL引用指向Java虚拟机中的一个null对象，如果obj是一个局部引用或者全局引用，可以使用:

```c
(*env)->IsSameObject(env, obj, NULL)
```
或者

```c
obj == NULL
```
来判断是否指向null对象。<br>
弱全局引用的规则有所不同。NULL弱引用同样的指向一个null对象，但是`IsSameObject`的返回值有不一样的意义。**你可以使用`IsSameObject`来判断一个non-NULL弱引用是否指向仍然指向一个未被回收的对象**。假设wobj是一个non-NULL的弱引用，
```c
(*env)->IsSameObject(env, wobj, NULL)
```
在wobj所指向的对象已经被回收时返回`JNI_TRUE`, 否则返回`JNI_FALSE`。

## 释放引用
每一个JNI引用，除了指向的对象外，本身也占用一定的内存。JNI开发者在一定时间段内，需要注意程序所使用的引用数量。特别是，需要考虑程序最多能够创建的局部引用的数量，即使Java虚拟机会自动回收。短时间内创建大量的引用会导致内存溢出。

### 释放局部引用
在大多数情况下本地代码不需要考虑局部引用的释放。但是有时候JNI程序员需要显式的释放局部引用以免内存溢出。比如:
* 在实现一个本地方法调用时需要创建大量的局部引用。这会导致内部的JNI局部引用表溢出，需要显式的释放没用的局部引用。看个例子:
```c
for (i = 0; i < len; i++) {    
    jstring jstr = (*env)->GetObjectArrayElement(env, arr, i)
    ... 
    /* process jstr */    
    (*env)->DeleteLocalRef(env, jstr); 
}
```
* 写一个工具函数(utility function)，比如上文提到的`MyNewString`，在函数内及时的调用`DeleteLocalRef`释放局部引用。
* 所写的本地方法不会返回，可能在执行一个无限的批处理循环，这种情况下手动释放局部引用是必须的。
* 本地方法访问了一个大对象，创建了一个局部引用，接着该方法需要继续执行额外的计算才能返回，这时候，即使该局部引用不再被使用，也不会被GC回收。比如下面的代码，显式的调用`DeleteLocalRef`使得`lref`可以被GC回收，即使本地方法还没有返回。
```c
/* A native method implementation */ 
JNIEXPORT void JNICALL 
Java_pkg_Cls_func(JNIEnv *env, jobject this) {    
    lref = ...              
    /* a large Java object */    
    ...                     
    /* last use of lref */    
    (*env)->DeleteLocalRef(env, lref);
    lengthyComputation();   
    /* may take some time */    
    return;                 
    /* all local refs are freed */ 
}
```

### JDK1.2中管理局部引用
JDK1.2中提供了别的函数来管理局部引用的生命周期: `EnsureLocalCapacity`, `NewLocalRef`, `PushLocalFrame`, `PopLocalFrame`。<br>
**JNI规范中规定了Java虚拟机要确保本地方法能够创建至少16个局部引用**，这能够满足大部分本地方法的需求了。但是如果需要创建更多的局部引用，则需要调用`EnsureLocalCapacity`来确保足够的空间来创建局部引用。
```c
/* The number of local references to be created is equal to   the length of the array. */ 
if ((*env)->EnsureLocalCapacity(env, len)) < 0) {    
    ... 
    /* out of memory */ 
} 
for (i = 0; i < len; i++) {    
    jstring jstr = (*env)->GetObjectArrayElement(env, arr, i)；
    ... 
    /* process jstr */    
    /* DeleteLocalRef is no longer necessary */ 
}
```
当然，上面的代码需要耗费更多的空间。<br>
另外，可以使用`Push/PopLocalFrame`来创建作用范围层层嵌套的局部引用。上述例子可以修改为:
```c
#define N_REFS ... 
/* the maximum number of local references  used in each iteration */ 
for (i = 0; i < len; i++) {    
    if ((*env)->PushLocalFrame(env, N_REFS) < 0) {        
        ... 
        /* out of memory */    
    }    
    jstr = (*env)->GetObjectArrayElement(env, arr, i);    
    ... 
    /* process jstr */    
    (*env)->PopLocalFrame(env, NULL); 
}
```
`PushLocalFrame`会为特定数量的局部引用创建一个使用堆栈，而`PopLocalFrame`则会销毁堆栈顶端的引用。<br>
`Push/PopLocalFrame`函数提供了对局部引用更方便的管理，不用考虑运行过程中的每一个引用的生命周期。在上面的例子中，在处理`jstr`的过程过创建了局部引用，在调用`PopLocalFrame`后，这些引用会被销毁。<br>
`NewLocalObject`在工具函数中返回一个局部引用时非常有用。下文会介绍。<br>
JDK1.2支持命令行的选项: `-verbose:jni`。设置后JVM会报告局部引用数量超出预定值的错误。

### 释放全局引用
当不再需要全局引用时，需要调用`DeleteGlobalRef`来释放全局引用，否则GC不会回收对应的对象。当不再需要弱全局引用时，需要调用`DeleteWeakGlobalRef`释放弱全局引用。虽然GC还是能够回收该引用指向的对象，但是引用本身所占用的内存将不会被回收。

## 管理引用的规则
总结一下管理JNI引用的规则，目的是减少不必要的内存占用和对象残留。<br>
通常来说，有两种类型的本地方法，一种是直接使用的，另一种是被调用的工具函数。在第一种本地函数中，开发者需要注意的是循环中创建的局部引用和没有返回的函数中创建的局部引用。在本地方法中不要造成全局引用和弱引用的累积，这些引用在函数返回时不会自动的被回收。<br>
**在工具函数中注意不要留下任何未被释放的局部引用**，因为工具函数的调用具有不确定性，有可能会在短时间内被大量的调用，造成内存溢出。
* 当工具函数返回的是基本类型是，不能有任何的局部引用，全局引用和弱引用的累积。
* 当工具函数返回的是引用类型，那么除了返回的引用外，不能有任何的局部引用，全局引用和弱引用的累积。

允许工具函数创建和缓存一些全局引用和弱引用，因为这只会在第一次调用时创建这些引用。<br>
如果一个工具函数返回一个引用，需要详细的说明返回引用的类型，方便调用者更好的管理这些返回的引用。比如下面的例子, 我们需要知道`GetInfoString`返回的引用类型来正确的释放返回的JNI引用:
```c
while (JNI_TRUE) {    
    jstring infoString = GetInfoString(info);    
    ... /* process infoString */

    ??? /* we need to call DeleteLocalRef, DeleteGlobalRef,  or DeleteWeakGlobalRef depending on the type of  reference returned by GetInfoString. */ 
}
```
在JDK1.2中，`NewLocalRef`可以确保函数返回的一定是一个局部引用。看一个例子，例子在一个全局引用中缓存了一个频繁使用的字符串(CommonString), 
```c

jstring 
MyNewString(JNIEnv *env, jchar *chars, jint len) {    
    static jstring result;
    /* wstrncmp compares two Unicode strings */    
    if (wstrncmp("CommonString", chars, len) == 0) {        
        /* refers to the global ref caching "CommonString" */ 
        static jstring cachedString = NULL;        
        if (cachedString == NULL) {            
            /* create cachedString for the first time */ jstring cachedStringLocal = ... ;            
            /* cache the result in a global reference */
            cachedString = (*env)->NewGlobalRef(env, cachedStringLocal);        
        }        
        return (*env)->NewLocalRef(env, cachedString);    
    }
    ... 
    /* create the string as a local reference and store in  result as a local reference */    
    return result; 
}
```
上述代码中，`MyNewString`总会返回一个局部引用。<br>
`Push/PopLocalFrame`在管理局部引用的生命周期中是非常有用的。如果你在程序入口调用了`PushLocalFrame`, 那么必须在程序的出口中调用`PopLocalFrame`，确保局部引用被释放。建议使用这两函数。看个例子:
```c
jobject f(JNIEnv *env, ...) {    
    jobject result;    
    if ((*env)->PushLocalFrame(env, 10) < 0) {        
        /* frame not pushed, no PopLocalFrame needed */ 
        return NULL;    
    }    
    ...    
    result = ...;    
    if (...) {        
        /* remember to pop local frame before return */
        result = (*env)->PopLocalFrame(env, result);
        return result;    
    }    
    ...    
    result = (*env)->PopLocalFrame(env, result);    
    /* normal return */    
    return result; 
}
```
错误地使用`PopLocalFrame`可能会导致程序崩溃。<br>
上面的例子也说明了`PopLocalFrame`第二个参数的用法。`result`在当前frame里面被`PushLocalFrame`创建，`PopLocalFrame`在弹出堆栈顶端元素之前将第二个参数`result`转换为新的局部引用并储存在上一个frame中。



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