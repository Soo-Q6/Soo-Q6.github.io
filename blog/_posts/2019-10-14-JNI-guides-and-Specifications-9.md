---
title: (译)JNI程序规范和指南9——JNI如何使用本地库
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---
JNI的一个应用就是使用已有的本地库(C/C++)。一个典型的办法，就是创建一个包含一系列本地方法的类库。<br>

本文首先先介绍一个最直接的方式——一对一映射(one-to-one mapping)。然后介绍一种叫做共享stubs(shared stubs)的方法来简化封装任务。最后会介绍如何使用peer classes来封装本地数据结构。<br>
这些方法都是直接通过本地方法调用本地库，这样的话缺点就是本地方法会依赖于本地库。这样的应用只能运行在支持这个本地库的操作系统之上。一个更好的方法是声明一些与OS无关的本地方法。移植应用程序时只需要修改直接调用本地库的本地方法实现，而不需要修改应用程序和那些本地方法声明。

## 一对一映射
假设我们想要封装一个C标准库的`atol`函数:
```c
long atol(const char *str); 
```
`atol`函数解析字符串参数然后返回一个十进制的值。我们定义一个封装类:
```java
public class C {    
    public static native int atol(String str);    
    ... 
}
```
为了演示C++编程，本文将使用C++作为例子。C++实现如下:
```c++
JNIEXPORT jint JNICALL 
Java_C_atol(JNIEnv *env, jclass cls, jstring str) 
{    
    const char *cstr = env->GetStringUTFChars(str, 0);
    if (cstr == NULL) {        
        return 0; 
        /* out of memory */    
    }    
    int result = atol(cstr);    
    env->ReleaseStringUTFChars(str, cstr);    
    return result; 
}
```
例子使用`GetStringUTFChars`将Unicode格式的字符串转换为UTF-8字符串(因为十进制数字是ASCII字符)。<br>
下面演示一个更复杂的例子：传递一个结构体指针给C函数。假设我们要封装Win32的接口`CreateFile`:
```c
typedef void * HANDLE; 
typedef long DWORD; 
typedef struct {...} SECURITY_ATTRIBUTES;
HANDLE CreateFile(    
    const char *fileName,       // file name    DWORD
    desiredAccess,        // access (read-write) mode 
    DWORD shareMode,            // share mode    
    SECURITY_ATTRIBUTES *attrs, // security attributes  
    DWORD creationDistribution, // how to create    DWORD 
    flagsAndAttributes,   // file attributes    HANDLE 
    templateFile         // file with attr. to copy 
);
```
`CreateFile`支持一些Java文件API不支持的Win32特性。比如，`CreateFile`可以使用特定的访问模式和文件属性来打开Win32的管道和处理串行端口通信。<br>
创建一个封装类:
```java
public class Win32 {    
    public static native int CreateFile(        
        String fileName,          // file name        
        int desiredAccess,        // access (read-write) mode        
        int shareMode,            // share mode        
        int[] secAttrs,           // security attributes 
        int creationDistribution, // how to create     
        int flagsAndAttributes,   // file attributes    
        int templateFile);        // file with attr. to copy    
    ... 
}
```
由于在内存中的潜在的存储差异，我们不会将C结构体映射到Java的类，相反我们使用一个数组来存储C结构体`SECURITY_ATTRIBUTES`。调用函数也可以传递null来使用Win32默认的安全属性。本文不会介绍如何将结构体编码到一个int数组。下面是C++实现:
```c++
JNIEXPORT jint JNICALL Java_Win32_CreateFile(        
    JNIEnv *env,        
    jclass cls,        
    jstring  fileName,         // file name 
    jint desiredAccess, // access (read-write) mode 
    jint shareMode,            // share mode   
    jintArray secAttrs,        // security attributes  
    jint creationDistribution, // how to create        
    jint flagsAndAttributes,   // file attributes 
    jint templateFile)         // file with attr. to copy 
{    
    jint result = 0;    
    jint *cSecAttrs = NULL;    
    if (secAttrs) {        
        cSecAttrs = env->GetIntArrayElements(secAttrs, 0); 
        if (cSecAttrs == NULL) {            
            return 0; /* out of memory */        
        } 
    }
    char *cFileName = JNU_GetStringNativeChars(env, fileName);    
    if (cFileName) {        
        /* call the real Win32 function */        
        result = (jint)CreateFile(cFileName,
                        desiredAccess,
                        shareMode, 
                        (SECURITY_ATTRIBUTES *)cSecAttrs,
                        creationDistribution,
                        flagsAndAttributes,  
                        (HANDLE)templateFile);        
        free(cFileName);    
    }    
    /* else fall through, out of memory exception thrown */ 
    if (secAttrs) {        
        env->ReleaseIntArrayElements(secAttrs, cSecAttrs, 0);    
    }    
    return result; 
}
```
首先我们将存储在int数组中安全属性转换为jint数组。如果`secAttrs`为null，我们传递NULL给本地的`CreateFile`。接着我们调用`JNU_GetStringNativeChars`获取一个区域特定的文件名字符串。一旦获取成功就可以将参数传递给本地方法`CreateFile`。<br>
上面的例子展示了一个常见的方法来创建一个封装类。每一个本地方法(比如`CreateFile`)对应一个本地stub函数(比如`Java_Win32_CreateFile`)，然后映射到`Win32.CreateFile`。在一对一映射中，stub函数有以下作用：
1. stub作为JVM的本地方法之间的参数适配器。JVM希望本地方法能够遵循约定的命名方式以及传递两个额外的参数(`JNIEnv`指针和`this`指针)。
2. stub作为JVM和本地方法之间的数据格式转换器。


## 共享stubs
一对一映射需要为每一个本地方法写一个stub函数，如果需要封装的本地方法很多，那么工作量就很很大。下面介绍共享stubs来简化封装类的编写工作。<br>
共享stubs负责将调用者的请求分发给对应的本地方法，并复杂将调用者传递的参数类型转换为本地方法需要的类型。<br>
我们先介绍共享stubs是如何简化C.atol的实现的，然后再介绍shared stubs类`CFunction`:
```java
public class C {    
    private static CFunction c_atol =        
        new CFunction("msvcrt.dll", // native library name
                        "atol",       // C function name
                        "C");         // calling convention  
    public static int atol(String str) 
    {        
        return c_atol.callInt(new Object[] {str});    
    }    
    ... 
}
```
`C.atol`不再是一个本地方法，也就不需要一个stub函数了。`C.atol`使用`CFunction`类来定义，其内部实现了一个共享stubs。静态变量`C.c_atol`存储了一个对应msvcrt.dll库中的`atol`函数的`CFunciton`对象。一旦这个字段被初始化，调用`C.atol`只需要调用`c_atol.callInt`这个共享stubs。<br>
`CFunction`的类层次结构:
![]({{site.data.strings.blog_url}}JNI9-1.png)<br>

`CFunction`定义了一个指向C函数的指针，是`CPointer`的子类:
```java
public class CFunction extends CPointer {    
    public CFunction(String lib,     // native library name
                    String fname,   // C function name
                    String conv) {  // calling convention
        ...    
    }    
    public native int callInt(Object[] args);    
    ... 
}
```
`callInt`检查Object数组中每个元素的类型，转换格式，然后传递给底层C函数。然后返回int结果。当然`CFunction`也可以其他的函数来处理不同类型的返回结果。<br>
`CPointer`的定义:

```java
public abstract class CPointer {    
    public native void copyIn(             
        int bOff,     // offset from a C pointer
        int[] buf,    // source data             
        int off,      // offset into source             
        int len);     // number of elements to be copied  
        public native void copyOut(...);    
    ... 
}
```

`CPointer`是一个支持访问任意C指针的抽象类。比如，`copyIn`将`int`数组的内容复制到C指针指向的内存。这种操作方式可以控制地址空间内的任意内存，使用时一定要小心。像`CPointer.copyIn`这种本地方法可以直接操作C指针，是不安全的。<br>
`CMalloc`是`CPointer`的子类，指向C使用`malloc`分配的堆内存:

```java
public class CMalloc extends CPointer {    
    public CMalloc(int size) throws OutOfMemoryError { ... }
    public native void free();    
    ... 
}
```
`CMalloc`的构造函数在C堆中分配给定大小的内存块。`CMalloc.free`释放该内存。使用`CFunction`和`CMalloc`重新实现`Win32.CreateFile`：
```java
public class Win32 {    
    private static CFunction c_CreateFile = 
        new CFunction ("kernel32.dll", // native library name                       
                        "CreateFileA",    // native function
                        "JNI");           // calling convention
    public static int CreateFile(        
        String fileName,          // file name        
        int desiredAccess,        // access (read-write) mode        
        int shareMode,            // share mode        
        int[] secAttrs,           // security attributes 
        int creationDistribution, // how to create 
        int flagsAndAttributes,   // file attributes
        int templateFile)         // file with attr. to copy    
    {        
        CMalloc cSecAttrs = null;        
        if (secAttrs != null) {            
            cSecAttrs = new CMalloc(secAttrs.length * 4);
            cSecAttrs.copyIn(0, secAttrs, 0, secAttrs.length);        
        }        
        try {            
            return c_CreateFile.callInt(new Object[] {
                            fileName, 
                            new Integer(desiredAccess),
                            new Integer(shareMode), 
                            cSecAttrs, 
                            new Integer(creationDistribution),  
                            new Integer(flagsAndAttributes),   
                            new Integer(templateFile)});
        } finally {            
            if (secAttrs != null) { 
                cSecAttrs.free();            
            }        
        }    
    }    
    ... 
}
```
使用一个静态变量存储`CFunction`，通过kernel32.dll的`CreateFileA`访问Win32的`CreateFile`，另一个接口是`CreateFileW`，传入一个Unicode字符串作为文件名。`CFunction`遵循JNI调用转换，也就是标准的Win32调用转换(stdcall)。<br>
上面的代码中，首先在C的heap上面分配一个足够大的内存块儿来存储安全属性，然后把所有的参数打包成一个数组并通过`CFunction`这个shared stubs来调用底层的C函数`CreateFileA`。最后释放掉存储安全属性的C内存块。


## 一对一映射和共享stubs的对比
两种方式都可以用来构建本地库的封装类，也各有各的优缺点。<br>
共享stubs的优点是不再需要在本地代码中写大量的stub函数，但是使用共享stubs需要特别小心，因为这些相当与在Java中写C，破坏了Java的类型安全机制。<br>
一对一映射的优点是数据类型的转换效率更高，比如共享stubs中`CFunction.callInt`总是需要为每一个int参数创建一个Integer对象。另外，共享stubs最多只能处理一组预先定义的参数类型。


## 共享stubs的实现
上文将`CFunciton`, `CMalloc`, `CPointer`当作黑匣子，本节将介绍如何是用JNI来实现。
### `CPointer`类
抽象类`CPointer`包含一个64位的字段`peer`用来存储底层的C指针:
```java
public abstract class CPointer {    
    protected long peer;    
    public native void copyIn(int bOff, int[] buf, int off,int len);    
    public native void copyOut(...);    
    ... 
}
```
C++实现的`copyIn`：
```c++
JNIEXPORT void JNICALL 
Java_CPointer_copyIn__I_3III(JNIEnv *env, jobject self, jint boff, jintArray arr, jint off, jint len) 
{    
    long peer = env->GetLongField(self, FID_CPointer_peer); 
    env->GetIntArrayRegion(arr, off, len, (jint *)peer + boff); 
}
```
`FID_CPointer_peer`是提前计算好的`CPointer.peer`的字段ID。C++实现的函数名是为了区别重载函数。

### `CMalloc`类
增加了分配和释放内存的函数:
```java
public class CMalloc extends CPointer {   
    private static native long malloc(int size);    
    public CMalloc(int size) throws OutOfMemoryError { 
        peer = malloc(size);        
        if (peer == 0) {            
            throw new OutOfMemoryError();        
        }    
    }    
    public native void free();    
    ... 
}
```
构造函数调用`CMalloc.malloc`分配内存，C++实现的两个函数:
```c++
JNIEXPORT jlong JNICALL 
Java_CMalloc_malloc(JNIEnv *env, jclass cls, jint size) 
{    
    return (jlong)malloc(size); 
}
JNIEXPORT void JNICALL 
Java_CMalloc_free(JNIEnv *env, jobject self) 
{    
    long peer = env->GetLongField(self, FID_CPointer_peer);
    free((void *)peer); 
}
```

### `CFunction`类
这个类需要操作系统支持动态链接，依赖与特定CPU的汇编代码。以下是Win32/Intel x86架构的:
```java
public class CFunction extends CPointer {    
    private static final int CONV_C = 0;    
    private static final int CONV_JNI = 1;    
    private int conv;    
    private native long find(String lib, String fname);
    public CFunction(String lib,     // native library name
                    String fname,   // C function name 
                    String conv) {  // calling convention 
        if (conv.equals("C")) {            
            conv = CONV_C;        
        } 
        else if (conv.equals("JNI")) {            
            conv = CONV_JNI;        
        } else {            
            throw new IllegalArgumentException("bad calling convention");        
        }        
        peer = find(lib, fname);    
    }
    public native int callInt(Object[] args);    
    ... 
}
```
类中使用了一个`conv`字段来保存C函数的调用转换类型。`CFunction.find`的实现:
```c++
JNIEXPORT jlong JNICALL 
Java_CFunction_find(JNIEnv *env, jobject self, jstring lib,jstring fun) 
{    
    void *handle;    
    void *func;    
    char *libname;    
    char *funname;
    if ((libname = JNU_GetStringNativeChars(env, lib))) {
        if ((funname = JNU_GetStringNativeChars(env, fun))) {            
            if ((handle = LoadLibrary(libname))) { 
                if (!(func = GetProcAddress(handle, funname))) {                    
                    JNU_ThrowByName(env, "java/lang/UnsatisfiedLinkError", funname); 
                }            
            } else {                
                JNU_ThrowByName(env, "java/lang/UnsatisfiedLinkError", libname); 
            }            
            free(funname);        
        }        
        free(libname);    
    }    
    return (jlong)func; 
}
```
`CFunction.find`将库名和函数名转换为本地代码支持的字符串格式，然后调用Win32的API`LoadLibrary`和`GetProcAddress`定位C函数。<br>
`callInt`的实现:
```c++
JNIEXPORT jint JNICALL 
Java_CFunction_callInt(JNIEnv *env, jobject self,jobjectArray arr) 
{ 
#define MAX_NARGS 32    
    jint ires;    
    int nargs, nwords;    
    jboolean is_string[MAX_NARGS];    
    word_t args[MAX_NARGS];
    nargs = env->GetArrayLength(arr);    
    if (nargs > MAX_NARGS) {        
        JNU_ThrowByName(env, "java/lang/IllegalArgumentException", "too many arguments");
        return 0;    
    }
    // convert arguments    
    for (nwords = 0; nwords < nargs; nwords++) { 
        is_string[nwords] = JNI_FALSE;        
        jobject arg = env->GetObjectArrayElement(arr,nwords);
        if (arg == NULL) {            
            args[nwords].p = NULL;        
        } else if (env->IsInstanceOf(arg, Class_Integer)) {
            args[nwords].i = env->GetIntField(arg, FID_Integer_value);        
        } else if (env->IsInstanceOf(arg, Class_Float)) {
            args[nwords].f = env->GetFloatField(arg, FID_Float_value);        
        } else if (env->IsInstanceOf(arg, Class_CPointer)) {            
            args[nwords].p = (void *) env->GetLongField(arg, FID_CPointer_peer);        
        } else if (env->IsInstanceOf(arg, Class_String)) {  
            char * cstr = JNU_GetStringNativeChars(env, (jstring)arg);            
            if ((args[nwords].p = cstr) == NULL) { 
                goto cleanup; // error thrown 
            }            
            is_string[nwords] = JNI_TRUE;        
        } else {            
            JNU_ThrowByName(env, "java/lang/IllegalArgumentException", "unrecognized argument type");            
            goto cleanup;        
        }        
        env->DeleteLocalRef(arg); 
    }
    void *func = (void *)env->GetLongField(self,FID_CPointer_peer); 
    int conv = env->GetIntField(self,FID_CFunction_conv);
    // now transfer control to func.    
    ires = asm_dispatch(func, nwords, args, conv);
cleanup:    
    // free all the native strings we have created    
    for (int i = 0; i < nwords; i++) {        
        if (is_string[i]) {            
            free(args[i].p);        
        }    
    }    
    return ires; 
}
```
`word_t`表示一个机器字(machine word)，定义如下:
```c++
typedef union {    
    jint i;    
    jfloat f;    
    void *p; 
} word_t;
```
然后遍历数组，检查数组元素的类型:
* 如果元素是null，向C函数传递一个NULL指针。

* 如果参数是java.lang.Integer类的实例，取出其中的int值并传递给C函数。

* 如果元素是java.lang.Float类的实例，取出其中的float值传递给C函数。

* 如果元素是一个CPointer类的实例，取出其中的peer指针并传递给C函数。

* 如果参数是一个java.lang.String的实例，则把字符串转换成本地C字符串，然后传递给C函数。

* 否则的话，抛出IllegalArgumentException。<br>

我们需要将缓冲区的参数传递给C函数，这需要对C堆栈进行操作，以下是内联的汇编实现:
```c
int asm_dispatch(void *func,   // pointer to the C function
                int nwords, // number of words in args array                 
                word_t *args, // start of the argument data                 
                int conv)     // calling convention 0: C                               //                    1: JNI 
{    
    __asm {        
        mov esi, args        
        mov edx, nwords        
        // word address -> byte address        
        shl edx, 2        
        sub edx, 4        
        jc  args_done
        // push the last argument first    
    args_loop:        
        mov eax, DWORD PTR [esi+edx]        
        push eax        
        sub edx, 4        
        jge SHORT args_loop    
    args_done:        
        call func
        // check for calling convention        
        mov edx, conv        
        or edx, edx        
        jnz jni_call
        // pop the arguments        
        mov edx, nwords        
        shl edx, 2        
        add esp, edx    
    jni_call:        
        // done, return value in eax    
    } 
}
```
上述的汇编代码将参数复制到C堆栈，然后调用本地的C函数。然后判断调用约定(calling convention)是C还是JNI，执行不同的参数出栈操作。

> **调用约定(calling convention):**<br>
> C或C++自身并没有定义这些标识符，它们是编译器扩展，代表了某些调用约定。它们决定以何种顺序在何处放置参数，被调函数在何处能找到返回地址等等。


## Peer Classes
不论使用哪种方法构建一个封装类，都会面临数据结构传递的问题，回想一下`CPointer`的定义:
```java
public abstract class CPointer {    
    protected long peer;    
    public native void copyIn(int bOff, int[] buf, int off, int len);    
    public native void copyOut(...);    
    ... 
}
```
它包含了一个64位的字段指向本地的数据结构(上述例子指向C地址空间的一块内存)。`CPointer`的子类也继承了这个字段，比如`CMalloc`:
![]({{site.data.strings.blog_url}}JNI9-2.png)<br>

像`CPointer`,`CMalloc`这些直接和本地数据结构相关的类称为peer classes。你可以为不同的数据结构构建peer classes：
* 文件描述符
* socket描述符
* 窗口或UI元素<br>

### Java中的peer classes
JDK使用peer classes在内部实现了java.io, java.net和java.awt包。比如，java.io.FileDescriptor类包含了一个`fd`字段来表示文件描述符:
```java
// Implementation of the java.io.FileDescriptor class 
public final class FileDescriptor {    
    private int fd;    
    ... 
}
```
假设你要执行一个Java API不支持的文件操作，你可能会使用JNI来找到java.io.FileDescriptor类中的`fd`(JNI允许访问私有字段)，你以为就可以对该文件描述符执行本地的文件操作。但是这有两个问题:
* 首先这个方法依赖于私有字段`fd`，很难保证其他Java平台的java.io.FileDescriptor类实现是否还支持该文件描述符。
* 其次，直接操作`fd`字段可能会破坏内部的一致性。由于peer classes假定他们可以独占访问底层的数据结构，对数据结构的操作可能会引起不一致性。<br>

唯一的解决办法就是创建自己的peer classes来封装本地数据结构。上面的例子中你可以定义自己的文件操作符peer class和特定的文件操作然后再定义自己的Java API。

### 释放本地数据结构
peer classes是在Java中定义的，所以对象实例会被GC回收，你需要确保，对应的C的数据结构内存块也会被释放。`CMalloc`类中就有`free`函数显式的释放C内存：
```java
public class CMalloc extends CPointer {    
    public native void free();    
    ... 
}
```
有些程序员喜欢在peer class中加一个finalizer:
```java
public class CMalloc extends CPointer {    
    public native synchronized void free();    
    protected void finalize() {        
        free();    
    }    
    ... 
}
```
JVM在GC回收一个对象实例之前会调用`finalize`。即使你忘记了释放内存，`finalize`也会帮你释放C内存。<br>
可是，为了防止重复调用，你需要添加`synchronized`关键字，而且还需要修改`CMalloc.free`的实现:
```c++
JNIEXPORT void JNICALL 
Java_CMalloc_free(JNIEnv *env, jobject self) 
{    
    long peer = env->GetLongField(self, FID_CPointer_peer); 
    if (peer == 0) {        
        return; /* not an error, freed previously */    
    }    
    free((void *)peer);    
    peer = 0;    
    env->SetLongField(self, FID_CPointer_peer, peer);
}
```
我们这样设置`peer`字段：
```c++
peer = 0; 
env->SetLongField(self, FID_CPointer_peer, peer);
```
而不是这样:
```c++
env->SetLongField(self, FID_CPointer_peer, 0); 
```
是因为C++编译器会将`0`认为是一个32位的整数。<br>

定义finalizer是一个很好的保障，但是不能作为释放C内存的主要方式: C数据结构可能会占用更多的内存而peer class中的只是一个64位的字段，GC会认为占用内存太小而不会及时回收它；另外，定义了finalizer的类在对象创建和回收时效率要差一点。<br>

你可以不必创建finalizer，只要你确保C内存会被释放，特别是异常发生时，注意下面的例子:
```java
CMalloc cptr = new CMalloc(10); 
try {    
    ... // use cptr 
} finally {    
    cptr.free(); 
}
```
`finally`确保了发生异常时，C内存也会被释放。

### peer实例的反向指针
上文我们已经介绍过，peer classes一般会包含一个私有字段储存一个指向C数据结构的指针。有些情况下，在C数据结构中添加一个peer实例的引用是很有用的。比如，需要调用该peer实例的回调方法。<br>
假设我们我创建一个UI组件`KeyInput`，用户点击之后，操作系统调用C++方法`key_press`，该组件的C++实现`key_input`从中收到一个事件，然后通过触发`keyPressed`告知`KeyInput`实例该事件的发生。下图是描述的上述过程:
![]({{site.data.strings.blog_url}}JNI9-3.png)<br>

`KeyInput`peer class定义:
```java
class KeyInput {    
    private long peer;    
    private native long create();    
    private native void destroy(long peer);    
    public KeyInput() {        
        peer = create();    
    }    
    public destroy() {        
        destroy(peer);    
    }    
    private void keyPressed(int key) {        
        ... /* process the key event */    
    } 
}
```
`create`本地方法实现创建了一个`key_put`结构体。使用结构体是为了和Java中的类相区别(避免混淆):
```c++
// C++ structure, native counterpart of KeyInput struct 
key_input {    
    jobject back_ptr;         // back pointer to peer instance 
    int key_pressed(int key); // called by the operating system 
};
JNIEXPORT jlong JNICALL 
Java_KeyInput_create(JNIEnv *env, jobject self) 
{    
    key_input *cpp_obj = new key_input(); 
    cpp_obj->back_ptr = env->NewGlobalRef(self);    
    return (jlong)cpp_obj; 
}
JNIEXPORT void JNICALL 
Java_KeyInput_destroy(JNIEnv *env, jobject self, jlong peer)
{    
    key_input *cpp_obj = (key_input*)peer; 
    env->DeleteGlobalRef(cpp_obj->back_ptr);    
    delete cpp_obj;    
    return; 
}
```
`Java_KeyInput_create`创建结构体并初始化`back_ptr`字段为`KeyInput`的全局引用。`Java_KeyInput_destroy`释放全局引用和创建的结构体。`KeuInput`调用`create`来初始化这一过程:
![]({{site.data.strings.blog_url}}JNI9-4.png)<br>

当用户点击之后，操作系统调用`key_input::key_pressed`:
```c++
// returns 0 on success, -1 on failure 
int key_input::key_pressed(int key) {    
    jboolean has_exception;    
    JNIEnv *env = JNU_GetEnv();    
    JNU_CallMethodByName(env, 
                        &has_exception,
                        java_peer,
                        "keyPressed",
                        "()V",                         
                        key);    
    if (has_exception) {        
        env->ExceptionClear();        
        return -1;    
    } else {        
        return 0;    
    } 
}
```

最后再讨论一下如何避免内存泄漏的问题，假定使用finalizer的方式:
```java
class KeyInput {    
    ...    
    public synchronized destroy() {        
        if (peer != 0) {            
            destroy(peer);            
            peer = 0;        
        }    
    }    
    protect void finalize() {        
        destroy();    
    } 
}
```

上述代码是不会执行的，`KeyInput`实例不会被GC回收(除非你手动调用`destroy`)，原因就是创建了一个`KeyInput`的JNI全局引用，这会阻止GC回收。解决办法就是使用弱全局引用:
```c++
JNIEXPORT jlong JNICALL 
Java_KeyInput_create(JNIEnv *env, jobject self) 
{    
    key_input *cpp_obj = new key_input();
    cpp_obj->back_ptr = env->NewWeakGlobalRef(self);
    return (jlong)cpp_obj; 
}
JNIEXPORT void JNICALL 
Java_KeyInput_destroy(JNIEnv *env, jobject self, jlong peer)
{    
    key_input *cpp_obj = (key_input*)peer;
    env->DeleteWeakGlobalRef(cpp_obj->back_ptr);    
    delete cpp_obj;    
    return; 
}
```