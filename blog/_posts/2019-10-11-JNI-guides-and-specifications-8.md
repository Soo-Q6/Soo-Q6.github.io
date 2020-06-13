---
title: JNI程序规范和指南8——JNI的其他特征
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---

我们已经讨论过在编写本地方法和嵌入JVM时JNI的特征，接下来我们会介绍剩下的JNI特征。

## JNI和线程
JVM支持在同一地址空间下同时执行多个线程，这使得程序变得更加复杂。多个线程可能会同时访问同一个对象，同一个文件——或者说同一个资源。<br>
要弄清楚下面的内容，你需要熟悉多线程编程，你应该了解如何使用多线程，如何同步访问共享资源。

### 限制
在多线程环境中使用本地方法需要注意一些限制，比如:
* `JNIEnv`指针只在一个线程内有效，不能传递给其他线程。不同线程调用同一本地方法传递的是不同的`JNIEnv`指针。
* 局部引用只在创建的线程内有效。在有多线程访问局部引用时需要转换为全局引用。<br>

### 监视器的入口和出口
监视器是Java平台的基本同步机制。每一个对象可以动态的关联一个监视器。JNI允许使用这些监视器进行同步，从而实现与Java编程语言中的同步块等效的功能：
```java
synchronized (obj) {    
  ...                   // synchronized block 
}
```
JVM会保证在执行这个block中任何代码之前获取到该对象的监视器，这样保证了在任何时候最多只有一个线程拥有这些监视器并执行这个block里的代码。其他线程在获取监视器之前会被阻塞。<br>
JNI函数提供了相同的同步能力，你可以使用`MonitorEnter`获取监视器并`MonitorExit`放弃监视器:
```c
if ((*env)->MonitorEnter(env, obj) != JNI_OK) {    
  ... /* error handling */ 
} 
...     /* synchronized block */ 
if ((*env)->MonitorExit(env, obj) != JNI_OK) {    
  ... /* error handling */ 
};
```
执行上述代码，当前线程首先得进入`obj`对应的监视器才能执行同步块中的代码。`MonitorEnter`需要一个`jobject`作为参数而且在其他线程已经进入该监视器时阻塞。而线程在没有获取到监视器的情况下调用`MonitorExit`会报错，抛出`IllegalMonitorStateException`异常。我们需要对监视器操作进行可能的错误检测(当前线程可能没有足够的内存执行监视器操作)。<br>
在异常处理时要注意监视器操作的配对:
```c
if ((*env)->MonitorEnter(env, obj) != JNI_OK) ...; 
... 
if ((*env)->ExceptionOccurred(env)) {    
  ... /* exception handling */    
  /* remember to call MonitorExit here */    
  if ((*env)->MonitorExit(env, obj) != JNI_OK) ...; 
} 
... /* Normal execution path. */
if ((*env)->MonitorExit(env, obj) != JNI_OK) ...;
```
调用`MonitorExit`失败可能会导致死锁。通过和上文中Java的同步代码对比你会发现Java的实现要简单很多。所以，建议在Java中完成同步操作。

### 监视器的等待和唤醒
Java API提供了其他的线程同步方法:`Object.wait`, `Object.notify`, `Object.notifyAll`。由于监视器的等待唤醒操作的性能要求不高，所以没有提供JNI函数，本地代码可以通过JNI调用Java API的方式来实现:
```c
/* precomputed method IDs */ 
static jmethodID MID_Object_wait; 
static jmethodID MID_Object_notify; 
static jmethodID MID_Object_notifyAll;
void 
JNU_MonitorWait(JNIEnv *env, jobject object, jlong timeout) 
{    
  (*env)->CallVoidMethod(env, object, MID_Object_wait, timeout); 
}
void 
JNU_MonitorNotify(JNIEnv *env, jobject object) 
{    
  (*env)->CallVoidMethod(env, object, MID_Object_notify); 
}
void 
JNU_MonitorNotifyAll(JNIEnv *env, jobject object) 
{    
  (*env)->CallVoidMethod(env, object, MID_Object_notifyAll);
}
```
假设它们的方法ID已经被缓存。

### 在任意地方获取JNIEnv指针
前文我们提到过，`JNIEnv`只在当前的线程有效，一般作为本地方法的第一个参数被传进来，但是当前线程中那些不是直接从虚拟机中被调用的本地方法是不能从参数传入的(操作系统的回调函数就不能以参数的形式传递)。那么如何从任意的代码中获取`JNIEnv`指针呢？<br>
你可以通过“调用接口”`AttachCurrentThread`获取`JNIEnv`指针:
```c
JavaVM *jvm; /* already set */
f() 
{    
  JNIEnv *env;    
  (*jvm)->AttachCurrentThread(jvm, (void **)&env, NULL);
  ... 
  /* use env */ 
}
```
如果当前线程已经附着到虚拟机，`AttachCurrentThread`就会返回属于当前线程的`JNIEnv`指针。<br>

获取`JavaVM`指针的方法有很多：在创建虚拟机时对其进行记录，使用`JNI_GetCreatedJavaVMs`查询创建的虚拟机，调用JNI函数`GetJavaVM`或定义`JNI_OnLoad`处理程序。**与`JNIEnv`指针不同，`JavaVM`指针在多个线程中保持有效，因此可以将其缓存在全局变量中**。<br>

在JDK 1.2以上版本中提供了一个新的“调用接口”`GetEnv`，提供了检测当前线程是否已经附着到JVM中并返回对应的JVM。如果线程已经附加到JVM，那么`GetEnv`和`AttachCurrentThread`效果是一样的。

### 匹配线程模型
假设一个本地方法会在多线程中访问一个全局资源。本地代码应该调用JNI函数中的`MonitorEnter`, `MonitorExit`还是使用本地主机提供的线程同步方法？同样的，如果本地方法需要创建一个线程，是调用JNI方法创建`java.lang.Thread`对象然后调用`Thread.start`还是选择本地主机环境中提供的创建线程的方法？<br>
答案是，如果JVM支持与当前代码所使用的线程模型相匹配的线程模型，则所有这些方法都可以使用。线程模型指示了系统在系统调用中实现的必需的线程操作，例如调度，上下文切换，同步和阻塞。在**本地线程模型**中，操作系统管理所有必要的线程操作。另一方面，在**用户线程模型**中，应用程序代码实现线程操作。例如，Solaris上JDK和Java 2 SDK发行版附带的“Green Thread”模型使用ANSI C函数`setjmp`和`longjmp`来实现上下文切换。<br>
JNI程序员必须注意线程模型。如果JVM以及本地代码对线程和同步的定义不同，则使用本地代码的应用程序可能无法正常运行。例如，本机方法可以在其自己的线程模型中的同步操作中被阻止，但是在具有不同线程模型的JVM可能不知道执行该方法的线程被阻止了。应用程序出现死锁。<br>
本地代码和JVM只用相同的线程模型就叫做线程模型匹配。如果JVM使用本地线程支持，则本地代码可以自由的调用主机环境中的线程相关的方法。但是如果JVM是借助其他的用户线程模型，则本地代码需要先链接相同的线程库或者不使用线程相关的方法。<br>
大多数JVM仅支持JNI本地代码的特定线程模型。支持本地线程模型是最灵活的，本地线程模型在给定的主机环境中通常是首选的。依赖于特定用户线程模型的JVM可能会严重受限于它们可以使用的本地代码类型。<br>
一些虚拟机实现可能支持许多不同的线程模型。更加灵活的JVM类型甚至可以允许你在JVM内部使用自定义线程模型，从而确保虚拟机实现可以与本地代码一起使用。在着手可能需要本地代码的项目之前，你应该查阅JVM随附的文档以了解线程模型的限制。

## 国际化代码
在编写多语言环境的代码时要特别的小心。JNI提供了完整的权限使用JAVA平台的国际化特性。我们将以字符串转换为例，因为文件名和消息在许多语言环境中都可能包含非ASCII字符。<br>
JVM使用Unicode格式表示字符串。尽管有些平台(比如Windows NT)支持Unicode，但是大多数还是使用语言环境相关的编码。<br>
不要使用`GetStringUTFCHars`和`GetStringUTFRegion`来进行`jstirng`和区域特定的字符串之间的转换，除非该区域的字符串使用UTF-8的编码。UTF-8在表示传递给JNI的参数(名字和描述符)时是有用的，但是在表示地区特定的文件名时就不合适了。

### 从本地字符串中创建`jstring`
使用`String(byte[] bytes)`将本地字符串转化成`jstring`。下面的工具函数将区域特定的本地字符串转化为`jstring`:
```c
jstring JNU_NewStringNative(JNIEnv *env, const char *str) 
{    
  jstring result;    
  jbyteArray bytes = 0;    
  int len;    
  if ((*env)->EnsureLocalCapacity(env, 2) < 0) {
    return NULL; 
    /* out of memory error */    
  }    
  len = strlen(str);    
  bytes = (*env)->NewByteArray(env, len);    
  if (bytes != NULL) {        
    (*env)->SetByteArrayRegion(env, bytes, 0, len, (jbyte *)str); 
    result = (*env)->NewObject(env, Class_java_lang_String, MID_String_init, bytes);        
    (*env)->DeleteLocalRef(env, bytes);        
    return result;    
  } 
  /* else fall through */    
  return NULL; 
}
```
该方法创建了一个byte数组，将本地的C字符串复制到该数组，最后调用`String(byte[] bytes)`构造函数创建`jstring`对象。`Class_java_lang_String`是java.lang.String类的全局引用，`MID_String_init`是String构造函数的方法ID。

### 将`jstring`转换为本地字符串
使用`String.getBytes`将`jstring`转换为合适的本地编码的字符串。下面的工具函数将`jstring`转换为一个区域特定的C字符串:
```c
char *JNU_GetStringNativeChars(JNIEnv *env, jstring jstr) 
{    
  jbyteArray bytes = 0;    
  jthrowable exc;    
  char *result = 0;    
  if ((*env)->EnsureLocalCapacity(env, 2) < 0) {
    return 0; 
    /* out of memory error */    
  }    
  bytes = (*env)->CallObjectMethod(env, jstr, MID_String_getBytes);    
  exc = (*env)->ExceptionOccurred(env);    
  if (!exc) {        
    jint len = (*env)->GetArrayLength(env, bytes); 
    result = (char *)malloc(len + 1);        
    if (result == 0) {            
      JNU_ThrowByName(env, "java/lang/OutOfMemoryError", 0);
      (*env)->DeleteLocalRef(env, bytes);            
      return 0;        
    }        
    (*env)->GetByteArrayRegion(env, bytes, 0, len, (jbyte *)result);        
    result[len] = 0; 
    /* NULL-terminate */    
  } else {        
    (*env)->DeleteLocalRef(env, exc);    
  }    
  (*env)->DeleteLocalRef(env, bytes);    
  return result; 
}
```
该方法传递一个java.lang.String引用给`String.getBytes`然后将byte数组的内容复制给C数组。`MID_String_getBytes`是方法ID。

## 注册本地方法
在执行本地方法之前需要先加载本地库并链接本地方法:
1. `System.loadLibrary`加载本地库
2. JVM在某一个本地库中定位本地方法。比如`Foo.g`对应`Java_Foo_g`<br>

本节将介绍另外的方法来实现本地方法的定位。JNI可以手动的链接本地方法：
```c
JNINativeMethod nm; 
nm.name = "g"; /* method descriptor assigned to signature field */ 
nm.signature = "()V"; 
nm.fnPtr = g_impl; 
(*env)->RegisterNatives(env, cls, &nm, 1);
```
以上方法注册了一个`Foo.g`实现`g_impl`：
```c
void JNICALL g_impl(JNIEnv *env, jobject self);
```
`g_impl`不需要遵循JNI命名规则，也不需要使用`JNIEXPORT`。使用`RegisterMatives`有以下好处:
* 在更方便和有效率的注册大量的JNI函数
* 可以多次调用`RegisterNatives`，运行时更新方法的实现
* `RegisterNatives`在本地应用嵌入了一个JVM并且需要链接本地方法时很有用。JVM不能自动查找该本地方法的实现，因为他只能在本地库中查找而不能搜索应用本身。

## 加载和卸载处理程序
加载卸载程序允许本地库提供两个方法：一个在`System.loadLibrary`时调用，一个在JVM卸载本地库时调用。

### `JNI_OnLoad`处理程序
JVM加载本地库时会搜索以下函数:
```c
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *jvm, void *reserved); 
```
可以在刚方法中调用任何函数，一般用于缓存`JavaVM`指针，类引用或者方法ID:
```c
JavaVM *cached_jvm; 
jclass Class_C; 
jmethodID MID_C_g;
JNIEXPORT jint JNICALL 
JNI_OnLoad(JavaVM *jvm, void *reserved) 
{    
  JNIEnv *env;    
  jclass cls;    
  cached_jvm = jvm;  /* cache the JavaVM pointer */
  
  if ((*jvm)->GetEnv(jvm, (void **)&env, JNI_VERSION_1_2)) {        
    return JNI_ERR; /* JNI version not supported */    
  }    
  cls = (*env)->FindClass(env, "C");    
  if (cls == NULL) {        
    return JNI_ERR;    
  }    
  /* Use weak global ref to allow C class to be unloaded */ 
  Class_C = (*env)->NewWeakGlobalRef(env, cls);    
  if (Class_C == NULL) {        
    return JNI_ERR;    
  }    
  /* Compute and cache the method ID */    
  MID_C_g = (*env)->GetMethodID(env, cls, "g", "()V"); 
  if (MID_C_g == NULL) {        
    return JNI_ERR;    
  }    
  return JNI_VERSION_1_2; 
}
```
缓存`JavaVM`指针可以实现一个工具函数，允许当前线程获取`JNIEnv`指针。
```c
JNIEnv *JNU_GetEnv() 
{    
  JNIEnv *env;    
  (*cached_jvm)->GetEnv(cached_jvm, (void **)&env, JNI_VERSION_1_2);    
  return env; 
}
```

### `JNI_OnUnload`处理函数
卸载本地库的规则:
* JVM将每个本地库与调用`System.loadLibrary`的类C的类加载器L关联。
* 在JVM发现不再需要类加载器L，那么就调用`JNI_OnUnload`然后卸载本地库。由于类加载器和它创建的所有的类相关，这意味着类C也可以被释放。
* `JNI_Onload`处理程序在finalizer中运行，并且可以由java.lang.System.runFinalization同步调用，也可以由虚拟机异步调用。

下面是一个例子：
```c
JNIEXPORT void JNICALL 
JNI_OnUnload(JavaVM *jvm, void *reserved) 
{    
  JNIEnv *env;    
  if ((*jvm)->GetEnv(jvm, (void **)&env, JNI_VERSION_1_2)) {        
    return;    
  }    
  (*env)->DeleteWeakGlobalRef(env, Class_C);    
  return; 
}
```
我们不需要删除方法ID`MID_C_g`，因为虚拟机在卸载其定义类C时会自动回收代表C方法ID所需的资源。<br>
现在我们准备解释为什么我们在弱全局引用而不是全局引用中缓存C类。 全局引用将使C保持活动状态，进而使C的类加载器保持活动状态。本地库与C的类加载器L相关联，因此不会卸载本地库，也不会调用`JNI_OnUnload`。<br>
由于`JNI_OnUnload`运行在一个未知的线程上下文，所以为了避免死锁，最好避免使用复杂的同步操作。`JNI_OnUnload`最好只进行资源的释放。<br>
当加载库的类加载器和该类加载器定义的所有类不再活跃时，`JNI_OnUnload`处理程序将运行。`JNI_OnUnload`处理程序不得以任何方式使用这些类。在上面的`JNI_OnUnload`定义中，不得执行任何假定Class_C仍引用有效类的操作。该示例中的`DeleteWeakGlobalRef`为弱全局引用释放了内存，但不以任何方式操纵所引用的类C。<br>

总而言之，要小心处理`JNI_OnUnload`，避免复杂的锁操作，注意在调用`JNI_OnUnload`时类已经被释放了。

## 反射机制的支持
反射通常是指在运行时处理语言级别的构造。例如，反射可以在运行时发现任意类对象的名称以及在类中定义的字段和方法的集合。通过java.lang.reflect包以及java.lang.Object和java.lang.Class类中的一些方法在Java中提供了反射支持。尽管总是可以调用相应的Java API来执行反射操作，但是JNI提供了以下功能，使本机代码中的频繁反射操作更加有效和便捷：
* ` GetSuperclass`返回给定类的父类
* `IsAssignableFrom`判断一个类是否能转化为另一个类
* `GetObjectClass`返回给定`jobject`引用的类
* `IsInstanceOf`检测一个`jobject`引用使用是一个给定类的实例
* `FromReflectedField`和`ToReflectedField`允许本地代码在字段ID和java.lang.reflect.Field对象间的转换
* `FromReflectedMethod`和`ToReflectedMethod`允许本地代码在方法ID和java.lang.reflect.Method对象之间的转换

## C++中的JNI
JNI提供了一些简单的接口给C++程序员，比如:
```c
//C++
jclass cls = env->FindClass("java/lang/String");
//C
jclass cls = (*env)->FindClass(env, "java/lang/String")
```
C和C++执行代码执行的结果是一样的，也没有性能上的差异。<br>
另外，jni.h还定义了一组虚拟C++类:
```c
// JNI reference types defined in C++     
class _jobject {};     
class _jclass : public _jobject {};     
class _jstring : public _jobject {};     
...     
typedef _jobject* jobject;     
typedef _jclass*  jclass;     
typedef _jstring* jstring;     
...
```

C++编译器能够在编译时识别传递一个`jobject`给`GetMethodID`：
```c
// ERROR: pass jobject as a jclass: 
jobject obj = env->NewObject(...); 
jmethodID mid =  env->GetMethodID(obj, "foo", "()V");
```
因为`GetMethodID`需要一个`jclass`作为参数，C++编译器会报错。在JNI的C类型定义中，`jclass`和`jobject`是一样的:
```c
typedef jobject jclass; 
```
所以C编译器不能识别。<br>

C++中添加的类型层次结构有时需要额外的转换。<br>
在C中:
```c
jstring jstr = (*env)->GetObjectArrayElement(env, arr, i); 
```
在C++中：
```c
jstring jstr = (jstring)env->GetObjectArrayElement(arr, i);
```