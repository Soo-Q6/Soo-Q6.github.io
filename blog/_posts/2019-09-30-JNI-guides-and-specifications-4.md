---
title: (译)JNI程序规范和指南4——字段和方法
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---

[上一篇](/blog/2019-09-29-JNI-guides-and-specifications-3)文章介绍了JNI如何访问基本类型和引用类型数据，本文将继续介绍如何访问任意对象的字段和方法。在本地代码中调用Java中实现的方法，也就是常说的回调函数**callback**。<br>
本文会介绍如何使用JNI函数访问对象的字段和调用回调函数，后面也会介绍如何使用缓存来使得对象操作更加简便和有效率。在文章的最后，还会讨论以下Java调用C/C++方法，C/C++访问Java对象字段和调用callback的性能。

## 访问字段
Java支持两种字段，对象字段(instance fields)和静态字段(static fields)。
> The JNI provides functions that native code can use to get and set instance ﬁelds in objects and static ﬁelds in classes.

让我们看一个简单的例子(访问对象字段):

```java
import java.lang.*;
class InstanceFieldAccess {    
    private String s;
    private native void accessField();    
    public static void main(String args[]) {        
        InstanceFieldAccess c = new InstanceFieldAccess();       
         c.s = "abc";        
         c.accessField();        
         System.out.println("In Java:");        
         System.out.println("  c.s = \"" + c.s + "\"");    
    }    
    static {        
        System.loadLibrary("InstanceFieldAccess");    
    } 
}
```
`InstanceFieldAccess`类定义了对象的字段`s`。`main`函数则创建了一个对象，为字段赋值，然后调用`InstanceFieldAccess.accessField`。该本地方法输出并修改该对象字段的值：

```c
#include <jni.h>
#include <stdio.h>
#include "InstanceFieldAccess.h"

JNIEXPORT void JNICALL 
Java_InstanceFieldAccess_accessField(JNIEnv *env, jobject obj) {    
    jfieldID fid;   
    /* store the field ID */    
    jstring jstr;    
    const char *str;
    /* Get a reference to obj’s class */    
    jclass cls = (*env)->GetObjectClass(env, obj);
    printf("In C:\n");
    /* Look for the instance field s in cls */    
    fid = (*env)->GetFieldID(env, cls, "s",  "Ljava/lang/String;");    
    if (fid == NULL) {        
        return; 
        /* failed to find the field */    
    }
    /* Read the instance field s */    
    jstr = (*env)->GetObjectField(env, obj, fid);    
    str = (*env)->GetStringUTFChars(env, jstr, NULL);    
    if (str == NULL) {        
        return; 
        /* out of memory */    
    }    
    printf("  c.s = \"%s\"\n", str);    
    (*env)->ReleaseStringUTFChars(env, jstr, str);
    /* Create a new string and overwrite the instance field */    
    jstr = (*env)->NewStringUTF(env, "123");    
    if (jstr == NULL) {        
        return; 
        /* out of memory */    
    }    
    (*env)->SetObjectField(env, obj, fid, jstr); 
}
```

运行结果如下：

```
In C:
  c.s = "abc"
In Java:
  c.s = "123"
``` 

### 访问对象对象字段的流程
本地方法访问对象的字段有两个步骤。首先，调用`GetFieldID`获取字段的ID：

```c
fid = (*env)->GetFieldID(env, cls, "s", "Ljava/lang/String;");
```

其中你需要通过`GetObjectClass`来获取`jobject obj`的类引用`cls`, 当你获得了字段的ID之后，就可以通过适当的JNI函数获得字段的值, 比如:

```c
jstr = (*env)->GetObjectField(env, obj, fid);
```

由于字符串和数组都属于对象，所以可以使用`GetObjectField`来访问字段。除此之外，JNI还支持`GetIntField`和`SetFloatField`。

### 字段描述符(field descriptors)
还记得前文说到`"Ljava/lang/String;"`来表示Java中的字段类型，这就是字段描述符，是由字段的声明类型决定的。比如`"I"`表示`int`,`"F"`表示`float`, `"D"`表示`double`, `"Z"`表示`double`等等。<br>
引用类型的描述符: 以`L`开头，接着是Java中的类型名(`.`换成`/`，最后`;`结尾)。<br>
数组类型的描述符：以`[`开头接着是元素的类型描述符。比如`[I`表示`int []`。<br>
可以使用**javap**工具获取.class文件中所有的类型描述符：

```
javap -s -p InstanceFieldAccess
```

### 访问静态字段
访问静态字段的方法和对象字段相似，看个例子:

```java
import java.lang.*;
class StaticFieldAccess {    
    private static int si;
    private native void accessField();   
     public static void main(String args[]) {        
         StaticFieldAccess c = new StaticFieldAccess();        
         StaticFieldAccess.si = 100;        
         c.accessField();        
         System.out.println("In Java:");       
          System.out.println("  StaticFieldAccess.si = " + si);    
    }    
    static {        
        System.loadLibrary("StaticFieldAccess");    
    } 
}
```
`StaticFieldAccess`包含了一个静态字段`int si`。在`StaticFieldAccess.main`中初始化了静态字段然后调用`StaticFieldAccess.accessField`输出并修改字段:

```c
#include <jni.h>
#include <stdio.h>
#include "StaticFieldAccess.h"

JNIEXPORT void JNICALL 
Java_StaticFieldAccess_accessField(JNIEnv *env, jobject obj) {    
    jfieldID fid;   
    /* store the field ID */    
    jint si;
    /* Get a reference to obj’s class */   
     jclass cls = (*env)->GetObjectClass(env, obj);
    printf("In C:\n");
    /* Look for the static field si in cls */    
    fid = (*env)->GetStaticFieldID(env, cls, "si", "I");    
    if (fid == NULL) {        
        return; 
        /* field not found */    
    }    
    /* Access the static field si */    
    si = (*env)->GetStaticIntField(env, cls, fid);    
    printf("  StaticFieldAccess.si = %d\n", si);    
    (*env)->SetStaticIntField(env, cls, fid, 200); 
}
```

输出结果:

```
In C:
  StaticFieldAccess.si = 100
In Java:
  StaticFieldAccess.si = 200
```
总结一下，访问静态字段和对象字段主要有两点差异:
* 获取字段ID的方法不同:`GetStaticFieldID`和`GetFieldID`
* 在访问字段的方法中(`GetStaticIntField`, `GetStaticIntField`)，静态字段传递的是类引用(class reference)而对象方法传递的是对象引用(object reference)

## 调用函数
Java有几种函数类型，对象函数(Instance methods), 静态方函数(static methods)和构造函数(constructors)。<br>
JNI支持很多方法来调用Java的回调函数。下面是一个C调用Java函数的例子:

```java
import java.lang.*;

class InstanceMethodCall {    
    private native void nativeMethod();    
    private void callback() {        
        System.out.println("In Java");    
    }    
    public static void main(String args[]) {        
        InstanceMethodCall c = new InstanceMethodCall();        
        c.nativeMethod();    
    }    
    static {       
        System.loadLibrary("InstanceMethodCall");    
    } 
}
```

本地代码实现:

```c
#include <jni.h>
#include <stdio.h>
#include "InstanceMethodCall.h"

JNIEXPORT void JNICALL 
Java_InstanceMethodCall_nativeMethod(JNIEnv *env, jobject obj) {    
    jclass cls = (*env)->GetObjectClass(env, obj);   
    jmethodID mid = (*env)->GetMethodID(env, cls, "callback", "()V");    
    if (mid == NULL) {        
        return; 
        /* method not found */    
    }    
    printf("In C\n");    
    (*env)->CallVoidMethod(env, obj, mid); 
}
```

运行结果:

```
In C
In Java
```

### 调用实例方法
`Java_InstanceMethodCall_nativeMethod`函数展示了如何调用一个对象的函数:
* 首先调用`GetMwthodID`遍历给定类的方法(根据返回类型和名字), 如果该方法不存在，则返回`NULL`并抛出`NoSuchMethodError`异常。
* 然后调用`CallVoidMethod`去执行该对象的方法。

除了`CallVoidMethod`，JNI还提供了其他的函数去执行Java中定义的函数。比如`CallIntMethod`, `CallObjectMethod`等等。另外还可以使用`Call<Type>Method`这类方法去调用Java中的API。

```
jobject thd = ...; 
/* a java.lang.Thread instance */ 
jmethodID mid; 
jclass runnableIntf =    (*env)->FindClass(env, "java/lang/Runnable"); 
if (runnableIntf == NULL) {    
  ... 
  /* error handling */ 
} 
mid = (*env)->GetMethodID(env, runnableIntf, "run", "()V"); 
if (mid == NULL) {    
  ... 
  /* error handling */ 
} 
(*env)->CallVoidMethod(env, thd, mid); 
... 
/* check for possible exceptions */
```

### 生成方法描述符
JNI使用类似与定义字段类型的描述符来定义函数的类型。一个函数描述符包含了形参类型和返回类型，形参类型在前并使用`()`括住，遵循函数中的排列顺序，并且多个形参之间没有分隔符。返回类型描述符紧跟其后。同样可以使用`javap`工具生成描述符。

### 调用静态方法
根据以下步骤在本地代码中调用Java定义的静态函数:
* 使用`GetStaticMethodID`获取函数的ID
* 将类，方法ID和参数传给`CallStatic<Type>Method`<br>

注意，调用静态方法传递的是类引用而实例方法则是传递对象的引用。在Java中你可以通过类直接调用静态方法，也可以通过一个new对象来调用(类Cls中有静态方法f，`Cls.f`和`obj=new Cls();obj.f`都是合法的)。但是在JNI中，本地方法只能通过类引用来调用静态方法。看个例子:<br>
Java代码:

```java
import java.lang.*;

class StaticMethodCall {    
    private native void nativeMethod();    
    private static void callback() {        
        System.out.println("In Java");    
    }    
    public static void main(String args[]) {        
        StaticMethodCall c = new StaticMethodCall();        
        c.nativeMethod();    
    }    
    static {        
        System.loadLibrary("StaticMethodCall");    
    } 
}
```

C代码：

```c
#include <jni.h>
#include <stdio.h>
#include "StaticMethodCall.h"

JNIEXPORT void JNICALL 
Java_StaticMethodCall_nativeMethod(JNIEnv *env, jobject obj) {    
    jclass cls = (*env)->GetObjectClass(env, obj);    
    jmethodID mid = (*env)->GetStaticMethodID(env, cls, "callback", "()V");    
    if (mid == NULL) {        
        return;  
        /* method not found */    
    }    
    printf("In C\n");    
    (*env)->CallStaticVoidMethod(env, cls, mid); 
}
```
### 调用父类的实例方法
你可以调用已经被重载过的父类的实例方法。JNI提供了`CallNonvirtual<Type>Method`这类方法。你需要按照以下的步骤进行调用:
* 调用`GetMethodID`从一个指向父类的引用中获取函数ID
* 传递对象实例(object)，父类引用(superclass)，函数ID和参数给`CallNonvirtual<Type>Method`

本地代码调用父类的实例方法是很少见到，Java中实现就很简单: `super.f()`。<br>
`CallNonvirtualVoidMethod`还可以用在调用父类构造函数上。

## 调用构造函数
JNI中调用构造函数的过程和调用实例方法很相似。为了获取构造函数的ID，需要传入`<init>`作为函数名，`V`作为函数返回类型。然后就可以通过`NewObject`等函数调用构造函数。以下例子在C中构造一个java.lang.String对象:

```c
jstring 
MyNewString(JNIEnv *env, jchar *chars, jint len) {    
  jclass stringClass;    
  jmethodID cid;    
  jcharArray elemArr;    
  jstring result;
  stringClass = (*env)->FindClass(env, "java/lang/String");    
  if (stringClass == NULL) {        
    return NULL; 
    /* exception thrown */ 
  }
  /* Get the method ID for the String(char[]) constructor */ 
  cid = (*env)->GetMethodID(env, stringClass, "<init>", "([C)V");  
  if (cid == NULL) {        
    return NULL; 
    /* exception thrown */    
  }
  /* Create a char[] that holds the string characters */ 
  elemArr = (*env)->NewCharArray(env, len);    
  if (elemArr == NULL) {        
    return NULL; 
    /* exception thrown */    
  }    
  (*env)->SetCharArrayRegion(env, elemArr, 0, len, chars);
  /* Construct a java.lang.String object */ 
  result = (*env)->NewObject(env, stringClass, cid, elemArr);
  /* Free local references */    
  (*env)->DeleteLocalRef(env, elemArr);    
  (*env)->DeleteLocalRef(env, stringClass);   
  return result; 
}
```

解释一下上面的例子, 首先`FindClass`返回java.lang.String的类引用。然后`GetMethodID`返回构造函数`String(char[])`的ID。`NewCharArray`创建缓冲区存储字符串。`NewObject`根据函数ID调用构造函数。NewObject函数需要的参数有：类的引用，构造方法的ID，构造方法需要的参数。`DeleteLocalRed`用来释放临时变量的资源，后面再做介绍。<br>
由于String很常用，所以JNI内置了更高效的`NewString`来替代上述JNI调用构造函数的过程。<br>
`CallNonvirtualVoidMethod`调用构造函数是可行的，不过本地函数首先需要调用`AllocObject`创建一个未初始化的对象(uninitialize object)：

```c
//替换result = (*env)->NewObject(env, stringClass, cid, elemArr);
result = (*env)->AllocObject(env, stringClass); 
if (result) {    
  (*env)->CallNonvirtualVoidMethod(env, result, stringClass, cid, elemArr);    
  /* we need to check for possible exceptions */    
  if ((*env)->ExceptionCheck(env)) {        
    (*env)->DeleteLocalRef(env, result);        
    result = NULL;    
  } 
}
```
使用`AllocObject`创建未初始化对象时一定要小心，注意不要对同一个对象调用多次构造函数。不过还是建议使用`NewObject`的方式，避免使用`AllocObject`/`CallNonvirtualVoidMethod`。

## 缓存字段和方法ID
获取字段和方法ID，需要根据名字和描述符进行检索，而检索的过程是很耗费资源的。下面介绍一个如何使用缓存技术来减少消耗。缓存字段和方法ID主要有两种方法，区别在缓存的时刻: 在字段和方法ID被使用的时候，或者定义字段和方法的类静态初始化的时候。<br>

### 在使用时缓存
在本地方法访问字段和方法的时候缓存它们的ID，下面的例子实现了将字段ID缓存在一个静态变量之中。

```c
JNIEXPORT void JNICALL 
Java_InstanceFieldAccess_accessField(JNIEnv *env, jobject obj) {
  static jfieldID fid_s = NULL; 
  /* cached field ID for s */
  jclass cls = (*env)->GetObjectClass(env, obj);    
  jstring jstr;    
  const char *str;
  if (fid_s == NULL) { 
    fid_s = (*env)->GetFieldID(env, cls, "s", "Ljava/lang/String;");if (fid_s == NULL) {            
      return; 
      /* exception already thrown */        
    }    
  }
  printf("In C:\n");
  jstr = (*env)->GetObjectField(env, obj, fid_s);    
  str = (*env)->GetStringUTFChars(env, jstr, NULL);    
  if (str == NULL) {        
    return; 
    /* out of memory */    
  }    
  printf("  c.s = \"%s\"\n", str);    
  (*env)->ReleaseStringUTFChars(env, jstr, str);
  jstr = (*env)->NewStringUTF(env, "123");    
  if (jstr == NULL) {        
    return; 
    /* out of memory */    
  }    
  (*env)->SetObjectField(env, obj, fid_s, jstr); 
}
```
`fid_s`缓存了`INstanceFieldAccess.s`的字段ID，初始化为`NULL`，第一次访问时被赋值。有人可能会说上述代码在多线程的时候会导致冲突，该缓存的值可能会被其他线程覆盖。但实际上影响不大，因为不同线程计算同一个字段的ID值是相等的。<br>
同样的方法缓存方法ID:
```c
jstring 
MyNewString(JNIEnv *env, jchar *chars, jint len) {    
  jclass stringClass;    
  jcharArray elemArr;    
  static jmethodID cid = NULL;    
  jstring result;
  stringClass = (*env)->FindClass(env, "java/lang/String");    
  if (stringClass == NULL) {        
    return NULL; 
    /* exception thrown */    
  }
  /* Note that cid is a static variable */    
  if (cid == NULL) {        
    /* Get the method ID for the String constructor */ 
    cid = (*env)->GetMethodID(env, stringClass, "<init>", "([C)V");
    if (cid == NULL) {            
      return NULL; 
      /* exception thrown */        
    }    
  }
  /* Create a char[] that holds the string characters */    
  elemArr = (*env)->NewCharArray(env, len);    
  if (elemArr == NULL) {        
    return NULL; 
    /* exception thrown */    
  }    
  (*env)->SetCharArrayRegion(env, elemArr, 0, len, chars);
  /* Construct a java.lang.String object */ 
  result = (*env)->NewObject(env, stringClass, cid, elemArr);
  /* Free local references */    
  (*env)->DeleteLocalRef(env, elemArr);    
  (*env)->DeleteLocalRef(env, stringClass);    
  return result; 
}
```

### 类的静态初始化过程缓存
上一种方法中，每一次都需要检查ID是否已经缓存。在很多情况下，在使用前就已经初始化并缓存ID是很方便的。Java虚拟机在调用类方法之前都会执行类的静态初始化(static initializer)过程。因此可以在静态初始化过程缓存字段和方法的ID。看个例子: <br>
```java
class InstanceMethodCall {    
  private static native void initIDs();    
  private native void nativeMethod();    
  private void callback() {        
    System.out.println("In Java");    
  }    
  public static void main(String args[]) {        
    InstanceMethodCall c = new InstanceMethodCall();        c.nativeMethod();    
  }    
  static {        
    System.loadLibrary("InstanceMethodCall");        
    initIDs();    
  } 
}
```
本地代码实现:
```c
#include <jni.h>
#include <stdio.h>
#include "InstanceMethodCall.h"

jmethodID MID_InstanceMethodCall_callback;
JNIEXPORT void JNICALL 
Java_InstanceMethodCall_initIDs(JNIEnv *env, jclass cls) {    
    MID_InstanceMethodCall_callback = (*env)->GetMethodID(env, cls, "callback", "()V"); 
}

JNIEXPORT void JNICALL 
Java_InstanceMethodCall_nativeMethod(JNIEnv *env, jobject obj) {  
    printf("In C\n");    
    (*env)->CallVoidMethod(env, obj, MID_InstanceMethodCall_callback); 
}
```
Java虚拟机在静态初始化`InstanceMethodCall`的时候执行了`initIDs`，缓存了方法ID，在`INstanceMethodCall.nativeMethod`中不再需要检索。

### 两种缓存方法的对比
如果开发者不能修改字段和方法所在类的源码，那么使用第一种方法(用时缓存)是合理的。比如，我们不能在java.lang.String中插入`initIDs`。<br>
用时缓存有几个缺点:
* 每次都需要检查，而且多线程时可能还会重复检索。
* ID在类unloaded之前才有效，要确保在本地方法还需要这些ID时，这个类不会unloaded或者reloaded([下一章](/blog/2019-10-02-JNI-guides-and-specifications-5)会介绍JNI如何通过创建一个类引用来防止类unloaded)。而静态初始化会在类reloaded的时候重新计算字段和方法的ID。<br>

综上，推荐在类的静态初始化时缓存字段和方法的ID。

## JNI中操作Java类字段和方法的性能
学习完使用缓存来提高效率之后，你可能会想了解JNI访问字段和方法的效率，native/Java回调对比起Java/native调用和Java/Java调用效率怎样？<br>
上述的疑问的取决与JVM执行JNI的效率。本文会通过分析调用本地方法回调和JNI操作字段和方法的过程来给出大概的概念。<br>
对比一下Java/native调用和Java/Java调用：

* Java/native调用相比于Java虚拟机内部的Java/Java调用来说多了一个调用转换的过程。当Java切换到本地方法时，Java虚拟机需要花费额外的操作来创建参数和栈帧。
* 内联Java/native调用要比内联Java/Java调用要更困难。

估计执行Java/native调用要比Java/Java调用慢2-3倍，而Java/native调用和native/Java回调效率差不多。<br>
JNI访问字段的花费主要取决于通过JNIEnv调用的成本。以释放一个对象来说，本地方法**必须**调用一个C函数来释放对象，这个C函数隔绝了本地代码和Java虚拟机的内部对象。*但是JNI访问字段的花销可以忽略不计。*


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