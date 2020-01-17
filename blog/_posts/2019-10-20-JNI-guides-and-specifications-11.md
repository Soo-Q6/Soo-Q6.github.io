---
title: (译)JNI程序规范和指南11——JNI设计概述
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---

本文主要内容是JNI的设计概述，必要时也会介绍底层实现的动机，为了让读者能够了解其中的trade-off。设计概述主要是作为JNI的主要概念，例如，`JNIEnv`指针，全局和局部引用，字段和方法ID等的规范。<br>


## 设计目标
JNI设计的最重要目标就是在不同操作系统的JVM中实现**二进制兼容性**，使得本地库不用重新编译就可以在不同的JVM上使用。
为此，JNI不能规定任何JVM实现的细节，因为JVM内部实现机制一直在更新，要保证JNI接口的稳定。<br>
JNI的第二个目标就是**效率**，这会和第一个目标冲突，有时候我们需要在平台无关性和效率之间做出选择。<br>
最后，JNI必须保证功能的完整性，必须提供足够多的JVM功能使得本地程序能够完成一些有用的任务。<br>
JNI不能只针对一款特定的JVM，而是要提供一系列标准的接口让程序员可以把他们的本地代码库加载到不同的JVM中去。

## 加载本地库
在应用调用本地方法之前，虚拟机必须先加载包含本地方法实现的本地库。

### 类加载器
本地库由**类加载器**定位。类加载器在JVM起着重要作用，比如加载class文件，定义类和接口，提供命名空间(namespace)分割机制以及加载本地库。本文假定你对类加载器有着基本的了解，所以不会介绍JVM加载和链接类的实现细节。每一个类或者接口对象都会与最初读取class文件并定义它们的类加载器(defining loader)相关联。两个类或者接口只有在名字和类加载器相同的情况下，才会是类型一致的。如下图所示，类加载器L1和L2都定义了类`C`，但两个类是不同的，因为包含了不同返回类型的方法`f`。<br>
![]({{site.data.string.blog_url}}JNI11-1.png)<br>

上图虚线表示类加载器之间的关系。一个类加载器可能需要请求其他的类加载器来为它加载类或者接口。比如上图中L1和L2就委托bootstrap class loader来加载java.lang.String类。委托机制允许系统类在不同的类加载器之间共享。这是必要的，因为如果程序和系统对java.lang.String有不同的理解会破坏类型安全机制。

### 类加载器和本地库
假设`f`在两个类`C`中都是本地方法，虚拟机使用"C_F"来定位他们的实现。为了保证正确的链接，每一个类加载器需要维护一个自己的本地库列表：
![]({{site.data.strings.blog_url}}JNI11-2.png)<br>

由于每一个类加载器都有自己的本地库列表，所以程序员只需要一个单一库(single library)来存储所需的全部本地方法，只要是被同一个加载器加载的类，都可以使用这些本地方法。当类加载器被GC回收时，相应的本地类也会被unload。

### 定位本地库
通过`System.loadLibrary`加载本地库。如下:
```java
package pkg; 
class Cls {     
    native double f(int i, String s);     
    static {         
        System.loadLibrary("mypkg");     
    } 
}
```
`System.loadLibrary`以本地库名"mypkg"为参数，虚拟机会根据操作系统的不同，将本地库名转换为不同的格式，比如Linux中的libmypkg.so和Windows中的mypkg.dll。<br>
JVM在启动时，会创建一个本地库的目录列表，这个列表随着OS的不同而不同，比如Windows下包含Windows的系统目录，应用的当前工作目录和PATH环境变量的目录。Linux则是LD_LIBRARY_PATH环境变量下的目录<br>
`System.loadLibrary`加载失败时会抛出`UNsatisfiedLinkError`异常，如果已经加载过本地库，则跳过。如果OS不支持动态链接，那么本地库prelink到虚拟机，此时`System.loadLibrary`不起作用。<br>
虚拟机为每一个类加载器维护一个以加载的本地库列表，如果有新加载的本地库，会根据下面的步骤来确定应该关联哪一个类加载器：
1. 确定`System.loadLibrary`的调用者
2. 确定定义调用者的类
3. 确定该类的类加载器<br>

下面的例子中，本地库foo和类`C`的类加载器相关联:
```java
class C {    
    static {        
        System.loadLibrary("foo");    
    } 
}
```
`ClassLoader.findLibrary`允许程序员指定给定类加载器的库加载策略，以平台无关的库名为参数:
* 返回`null`来指示虚拟机遵循默认的搜索路径，
* 或者返回本地库所在的绝对类路径。<br>

`ClassLoader.findLibrary`通常与`System.mapLIbraryMame`搭配使用，`System.mapLIbraryMame`将库名(比如"mupkg")映射到一个特定的名字(比如"mypkg.dll")。同时你也可以通过设置java.library.path复写默认的搜索路径:
```
java -Djava.library.path=c:\mylibs Foo
```


### 类型安全机制
JVM不允许一个JNI本地库被多个类加载器加载。多个类加载器加载同一个本地库时会抛出`UnsatisfiedLinkError`异常。这个限制是为了保证命名空间分割机制在本地库中同样有效。没有这个限制，在本地方法操作JVM时很容易造成不同类加载器的类和接口的混乱。考虑一个本地方法`Foo.f`在一个全局变量中缓存了定义它的类`Foo`：
```c
JNIEXPORT void JNICALL 
Java_Foo_f(JNIEnv *env, jobject self) 
{    
    static jclass cachedFooClass; /* cached class Foo */
    if (cachedFooClass == NULL) {        
        jclass fooClass = (*env)->FindClass(env, "Foo");
        if (fooClass == NULL) {            
            return; /* error */        
        }        
        cachedFooClass = (*env)->NewGlobalRef(env, fooClass);        
        if (cachedFooClass == NULL) {            
            return; /* error */        
        }    
    }    
    assert((*env)->IsInstanceOf(env, self, cachedFooClass));
    ... /* use cachedFooClass */ 
}
```
上述例子中，`Foo.f`是一个实例方法，`self`指向一个`Foo`对象。当有两个类加载器L1和L2都加载了类`Foo`，并且两个类`Foo`都和上面的`Foo.f`实现链接，此时assertion会失败。全局变量`cacheFooClass`会在第一个调用`f`方法的类`Foo`创建，后一个`f`调用会导致assertion出错。<br>
早期的JDK版本没有强制的在类加载器之间本地库的分割，这意味着两个类可能会和同一个本地库链接。这样的话，上面的例子就会有两个问题:
* 一个类可能会和一个已经被其他类加载器的类加载的本地库相链接。
* 本地方法很容易就会混淆不同类加载器的类。这会导致类型安全问题。<br>

### unload本地库
当GC回收了类加载器，相应的本地库也会被unload。


## 链接本地库
JVM在第一次调用本地方法时会链接它。JVM不应该过早的链接一个本地方法，这样会出现意想不到的连接错误，因为实现该本地方法的本地库还没有被加载。<br>
链接一个本地方法包含以下步骤:
* 确定定义该本地方法的类的类加载器
* 在该类加载器对应的本地库中搜索该本地方法的实现
* 建立内部数据结构使得后续的本地方法调用可以直接定向到本地方法的实现<br>

JVM通过下面的步骤来确定本地库中对应的本地方法名:
* “Java_”前缀
* 类的全名
* 分隔符“_”
* 方法名
* 如果是重载方法，后接两个下划线“__”，然后再接参数描述符<br>

JVM会遍历类加载器对应的本地库列表，根据上述的名字搜寻合适的本地方法实现。在搜索本地库时，先搜索短名(没有参数描述符的那种)，然后再搜索长名。长名只在出现重载的时候使用(如果是和非本地方法重载，也不需要长名)。比如下面的例子:
```java
class Cls1 {    
    int g(int i) { ... } // regular method    
    native int g(double d); 
}
```
JNI有一个简单的命名编码协议来所有的Unicode字符都能被编码为C函数名。使用“_”来取代“.”分割类全名。由于名字和类型描述符不会以数字开头，所以可以使用_0,...,_9作为转义序列。<br>

如果一个本地方法在多个本地库中被匹配，JVM会链接第一个被加载的本地库。如果没有匹配的，则会抛出`UnsatisfiedLinkError`异常。当然，也可以使用JNI函数`RegisterNatives`来注册一个类的本地函数，这个函数在静态链接中非常有用。<br>

## Calling convention
“调用转换”(calling convention)决定了一个本地函数如何接受参数并返回结果。这里没有标准，主要取决与本地语言和编译器。即使可能，JVM也很难适配多种调用转换机制。所以JNI要求在同一个系统环境下，调用转换是一样的，比如UNIX使用C调用转换，而Windows使用stdcall。<br>
如果调用函数西药遵循其他的调用转换机制，最好还是便有些一个中间层来实现。

## `JNIEnv`接口指针
本地代码通过指向函数表的`JNIEnv`接口指针来访问JVM的功能。

### `JNIEnv`接口指针的结构
`JNIEnv`是一个指向线程局部数据的指针，包含一个指向函数表的指针。在这个表中，每一个函数都在一个预定于的位置上面。`JNIEnv`指针的结构就像C++的虚拟函数表，或者Microsoft COM接口:
![]({{site.data.strings.blog_url}}JNI11-3.png)<br>

本地方法的实现以`JNIEnv`接口指针为第一个参数，JVM会保证同一个线程内调用的方法获取到同一个`JNIEnv`接口指针，不同线程之间，同一个方法会获取到不同的`JNIEnv`指针。但是`JNIEnv`指针间接指向的函数表是多线程共享的。<br>

`JNIEnv`指向一个线程内的局部数据结构的原因是有些平台不能有效支持线程内局部数据的存储访问。注意不要跨线程使用`JNIEnv`。

### 接口指针的好处
* 最重要的是，JNI函数表是作为函数表指针的方式传递给本地方法，这样本地库就不需要和一个特定的JVM链接。这使得本地库可以和多个JVM实现协作。
* JVM可以提供多个函数表。比如JVM提供两个版本的函数表，一个需要较多的错误检测，用于调试阶段；一个尽量简化错误检测，更高效，用于版本发布。
* 最后，支持多个JNI函数表，使得未来可以支持多个版本的JNIEnv-like接口。

## 传递数据
基本数据类型，比如整数，字符等，在JVM和本地代码之间直接复制传递，而对象，则是通过传递引用。每一个引用都包含一个指向JVM中对象的指针。本地代码不能直接使用指向对象的指针而是通过引用来访问。<br>

传递引用而不是指针使得JVM可以更灵活的管理对象。参考下图，当本地代码控制着一个引用的时候，JVM可能会执行垃圾回收操作，导致引用对应的对象被从一块内存移动到另一块内存，但这不会影响到本地代码。<br>
![]({{site.data.strings.blog_url}}JNI11-4.png)<br>

### 全局和局部引用
JNI为本地代码创建两种引用类型：全局引用和局部引用。局部引用生存周期局限于创建它的方法内部，方法返回后会被自动释放。全局引用知道手动释放后才失效。局部引用线程内有效。一个`NULL`引用指向JVM中的`null`对象，引用非`NULL`不能指向`null`对象。<br>

### 局部引用的实现
为了实现局部引用，JVM需要将对象的控制权交给本地代码，并为每一个对象的控制权传递创建一条记录(registry)。这条记录将一个不可移动的局部引用映射到一个对象指针。记录内的对象不能被GC回收。所有传递给本地代码的对象，包括作为JNI函数调用返回值的对象，都会自动的添加到记录里。当本地方法返回时，JVM会删除谢谢记录，并允许GC回收这个对象。下图说明了记录时如何创建和删除的。一个JVM frame对应一个本地方法，这个frame包含一个指向本地引用记录的指针。`D.f`调用本地方法`C.g`(其实现为`Java_C_g`)，JVM创建在访问`Java_C_g`之前一个本地引用记录，并在`Java_C_g`返回后删除该记录：
![]({{site.data.strings.blog_url}}JNI11-5.png)<br>

这里有不同的方式来实现该记录：栈，表，链表，哈希表。实现时可能使用引用计数来避免重复和冲突(不是必须)。以下这段话不理解，暂时留着：

> Local references can not be implemented faithfully by conservatively scanning the native stack. Native code may store local references into global or C heap data structures.<br>

### 弱全局引用
弱全局引用允许被GC回收，JVM回收了对象后，对应的弱全局引用也会被清除，可以通过`IsSameObject`和NULL引用对比来检测弱全局引用是否被清除。

## 访问对象
JNI提供了大量的函数来通过引用操作对象，这意味着本地方法可以不用关心不同JVM是如何存储对象的。尽管通过引用间接操作对象相对于直接操作指针会带来额外的开销，但是这是值得的。

### 访问基本类型数组
重复的访问对象中的基本类型数据的开销是不可接受的，比如整型数组和字符串。在操作向量和矩阵时，通过函数来访问每一个元素是十分低效的。<br>
一个解决办法是使用pinning的技术来告知JVM不要移动这个对象，这样本地代码就可以获取到指向这些元素的指针。这个方法意味着:
* GC需要支持pinning。很多JVM的实现是不支持pinning的，因为这会使得GC算法复杂化和造成内存碎片化。
* JVM需要在内存中连续存储数组。尽管这对大多数的数组来说是很正常的实现方式，但是Boolean类型的数组有packed和unpacked两种存储方式，packed使用1 bit来表示一个元素，而unpacked使用1 byte来表示一个元素。因此，依赖于Boolean数组特定存放方式的本地代码是不可移植的。<br>
JNI提供了一个折衷的解决方案:<br>
首先，JNI提供了一些函数(`GetIntArrayRegion`,`SetIntArrayRegion`等)在JVM和本地代码内存之间复制基本类型数组。<br>
其次，JNI提供了另外的函数(`GetIntArrayElement`等)来提供pinned版本的数组元素。这些函数可能会导致内存的重新何配和拷贝，是否拷贝取决于JVM的实现:
* GC支持pinning并且JVM中数组的存储方式和本地代码的需求一致，则不需要拷贝。
* 否则，数组会被拷贝到一个不可变(nonmovable)的内存块，比如C堆，并返回指向该拷贝的指针。<br>

当不再使用数组后，本地代码会调用另外一组函数(例如，`ReleaseIntArrayElement`)来通知JVM。这时，JVM会unpin数组或者把对复制后的数组的改变反映到原数组上然后释放复制后的数组。<br>
这种方式提供了很大的灵活性，让GC可以自由决定是复制还是pinning。<br>
另外JNI还提供了`GetPrimitivaArrayCritical`和`ReleasePrimitiveArrayCritical`这些“critical”的函数。正如[前文](/blog/2019-09-29-JNI-guides-and-specifications-3#jdk-12中的jni函数)所述，使用这些函数的限制很大，在“critical region”中不能调用其他的JNI函数，此时JVM会暂时的禁用GC同时本地代码能够直接访问JVM中的数组。<br>
JNI必须保证多线程可以同时访问同一个数组(对象)，比如，JVM可能会为每一个被pinned的数组维持一个计数器，以防止被其他线程unpinned。**值得注意的是，JNI不必为本地方法独占访问一个基本类型数组而使用锁，这意味着多线程同时更新一个数组是允许的，但这会导致意想不到的后果。**


### 字段和方法
JNI允许本地代码访问Java中定义的字段和方法，JNI通过名字和类型描述符来识别字段和方法。比如，要获取一个`cls`类的字段`i`，首先获取字段ID：
```c
jfieldID fid = env->GetFieldID(env, cls, "i", "I"); 
```
然后可以重复使用字段ID：
```c
jint value = env->GetIntField(env, obj, fid); 
```

字段和方法ID的声明周期取决于JVM中对应类或接口的生命周期。除非类或者接口被unload，否则ID是一直有效的。<br>
字段和方法可以定义在一个类或接口，也可以定义在父类或者间接父类。JVM规范中定义：**如果两个类或者接口定义了相同的字段或者方法，那么JNI解析的字段和方法ID是一样的**。比如，如果B定义了字段fld，C从B继承了fld，那么JNI可以从B和C中通过字段名fld获取到一样的字段ID。<br>
JNI不关心JVM内部是如何实现字段和方法ID的。<br>
JNI只有知道名字和类型才能访问字段和方法，而通过Java的反射(reflection)机制就可以在不知道名字和类型的情况下访问字段和方法。有时候在本地代码中反射也是很有用的，所以JNI提供了一组函数来支持Java的反射。这些函数包括了在JNI字段ID和java.lang.reflect.Field类之间的转换，方法ID和java.lang.reflect.Method类之间的转换。

## 错误和异常
JNI编程中出现的错误(errors)和JVM中产生的异常(exceptions)是不一样的。JNI编程时的错误一般是有误用JNI函数产生的，比如错误的给类操作方法`GetFieldID`转递一个对象。JVM抛出异常，比如本地代码通过JNI创建一个对象而内存不足时。<br>

### 不检查编程错误
JNI函数不检查编程错误。传递错误的参数给JNI函数会导致意想不到的结果。这样设计的原因是：
* 强制JNI函数去检查可能存在的错误会降低本地方法的性能。
* 很多情况下，没有足够的运行时类型信息去处理检查。<br>

大多数的C库函数不会检查编程错误，比如`printf`，通常会触发一个运行时错误，而不是返回一个错误码。强制C库函数检查所有可能存在的错误条件可能会导致重复进行错误检查，一次是用户代码，一次是库函数。<br>
虽然JNI规范没有要求JVM检查编程错误，但还是鼓励JVM去检查常见的错误，比如，在调式版本的JNI函数中会执行更过的错误检查。

### JVM异常
JNI不依赖于本地代码的异常处理机制。本地代码可以通过`Throw`和`throwNew`来向JVM抛出一个异常。一个未处理的异常会记录在当前线程中，和Java不一样的是，C代码出现异常不会立即中断代码。<br>
本地代码中没有标准的异常处理机制。所以JNI程序最好在调用可能抛出异常的JNI函数时手动处理异常：
* 本地方法可以选择立即返回，让代码中的异常向方法调用者抛出。
* 本地代码删除异常(`ExceptionClear`)并执行自己的异常处理方法。<br>

**异常抛出后一定要先处理异常再执行其他的JNI函数**，否则可能会导致意想不到的结果。下面是异常发生时能够安全调用的全部JNI函数:
```c
ExceptionOccurred 
ExceptionDescribe 
ExceptionClear 
ExceptionCheck

ReleaseStringChars 
ReleaseStringUTFchars 
ReleaseStringCritical 
Release<Type>ArrayElements 
ReleasePrimitiveArrayCritical 
DeleteLocalRef 
DeleteGlobalRef 
DeleteWeakGlobalRef 
MonitorExit
```
前四个方法是异常处理相关的，剩下的则是常规的释放JVM资源的方法。一般出现异常后都需要释放资源。<br>
