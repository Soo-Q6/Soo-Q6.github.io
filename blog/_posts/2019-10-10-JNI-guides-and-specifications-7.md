---
title: JNI程序规范和指南7——在本地代码中嵌入JVM
# image: /assets/img/blog/...
description: >
  这是一个关于JNI的系列文章。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [JNI]
---

本文主要介绍如何将JVM嵌入到本地代码中。JVM可以看作是一个本地库，本地程序可以链接这个库，使用“调用接口”(invocation interface)来加载JVM。实际上，JDK标准的启动器就是一段简单的链接了JVM的C代码，启动器解析命令行，加载JVM，然后通过“调用接口”(invocation interface)执行Java程序。

## 创建Java虚拟机
为了解释“调用接口”，让我们看一下下面一段加载JVM并执行`Prog.main`的C代码:
```java
public class Prog {    
    public static void main(String[] args) {
        System.out.println("Hello World " + args[0]);    
    } 
}
```
```c
#include <jni.h>
#include <stdlib.h>
#define PATH_SEPARATOR ';' /* define it to be ':' on Solaris */ 
#define USER_CLASSPATH "." /* where Prog.class is */
main() {    
    JNIEnv *env;    
    JavaVM *jvm;    
    jint res;    
    jclass cls;    
    jmethodID mid;    
    jstring jstr;    
    jclass stringClass;    
    jobjectArray args;
#ifdef JNI_VERSION_1_2    
    JavaVMInitArgs vm_args;    
    JavaVMOption options[1];    
    options[0].optionString = 
        "-Djava.class.path=" USER_CLASSPATH;
    vm_args.version = 0x00010002;    
    vm_args.options = options;    
    vm_args.nOptions = 1;    
    vm_args.ignoreUnrecognized = JNI_TRUE;    
    /* Create the Java VM */    
    res = JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args);
#else    
    JDK1_1InitArgs vm_args;    
    char classpath[1024];    
    vm_args.version = 0x00010001;    
    JNI_GetDefaultJavaVMInitArgs(&vm_args); 
    /* Append USER_CLASSPATH to the default system class path */    
    sprintf(classpath, "%s%c%s", 
        vm_args.classpath, PATH_SEPARATOR, USER_CLASSPATH); 
    vm_args.classpath = classpath;    
    /* Create the Java VM */    
    res = JNI_CreateJavaVM(&jvm, &env, &vm_args); 
#endif /* JNI_VERSION_1_2 */
    if (res < 0) {        
        fprintf(stderr, "Can't create Java VM\n");
        exit(1);    
    }    
    cls = (*env)->FindClass(env, "Prog");    
    if (cls == NULL) {        
        goto destroy; 
    }
    mid = (*env)->GetStaticMethodID(env, cls, "main", 
        "([Ljava/lang/String;)V");    
    if (mid == NULL) {        
        goto destroy;    
    }    
    jstr = (*env)->NewStringUTF(env, " from C!");    
    if (jstr == NULL) {        
        goto destroy;    
    }    
    stringClass = (*env)->FindClass(env, "java/lang/String");    
    args = (*env)->NewObjectArray(env, 1, stringClass, jstr);    
    if (args == NULL) {        
        goto destroy;    
    }    
    (*env)->CallStaticVoidMethod(env, cls, mid, args);
destroy:    
    if ((*env)->ExceptionOccurred(env)) {        
        (*env)->ExceptionDescribe(env);    
    }    
    (*jvm)->DestroyJavaVM(jvm); 
}
```
该代码有条件地编译一个初始化结构`JDK1_1InitArgs`，该结构体特定于JDK 1.1中的虚拟机实现。尽管JDK 1.2引入了称为`JavaVMInitArgs`的通用虚拟机初始化结构体，但它仍支持`JDK1_1InitArgs`。常量`JNI_VERSION_1_2`在JDK 1.2版中定义，在JDK 1.1版中未定义。<br>

在JDK 1.1中，C代码会调用`JNI_GetDefaultJavaVMInitArgs`来获取默认的JVM设置。`JNI_GetDefaultJavaVMInitArgs`会在`vm_args`中返回heap size, stack size和默认类路径等信息。然后我们将`Prog`类的路径添加到`vm_args.classpath`中。<br>

在JDK 1.2中，C代码会创建一个`JavaVMInitArgs`结构体，虚拟机的初始化参数都存储在`JavaVMOption`数组中。可以设置直接对应于Java命令行选项的常用选项(例如-Djava.class.path =.)和实现特定的选项(例如-Xmx64m)。将`ignoreUnrecognized`字段设置为`JNI_TRUE`会导致虚拟机忽略无法识别的实现特定选项。<br>

在设置完虚拟机初始化结构体后，C代码调用`JNI_CreateJavaVM`来加载JVM，`JNI_CreateJavaVM`有两个返回值：
* `jvm`接口指针指向新创建的JVM。
* 当前线程的JNIEnv接口指针env。本地代码通过env指针访问JNI函数。<vr>

当`JNI_CreateJavaVm`成功执行后，当前的本地线程会将程序控制权交给JVM，这时，它会像本地方法一样执行，然后通过JNI函数调用`Prog.main`。<br>
程序的最后，通过`DestroyJavaVM`来释放JVM。<br>
程序执行:<br>
```
cl -I"%JAVA_HOME%\include" -I"%JAVA_HOME%\include\win32" -c Prog.c
link Prog.obj "%JAVA_HOME%\lib\jvm.lib"
# 然后将jvm.dll所在路径添加到系统环境变量
Prog.exe
```
不知道为什么要将jvm.dll添加到系统环境变量而拷贝到当前路径就不行，但最后程序的结果:
```
Hello World  from C!
```

## 将JVM链接到本地代码
“调用接口”需要你将JVM的实现链接到比如invoke.c，如何链接JVM取决于本地程序是需要一个具体的JVM实现还是多个未知的实现方式的JVM一起工作。<br>

### 链接一个已知的JVM
本地程序可能需要和一个特定的JVM协作，那么你需要链接实现了JVM的本地库。比如，在Solaris中：
```
cc -I<jni.h dir> -L<libjava.so dir> -lthread -ljava invoke.c
```
在Windows中(还是参考上一节的方法吧...)：
```
cl -I<jni.h dir> -MD invoke.c -link <javai.lib dir>\javai.lib 
```
一旦链接成功，就可以执行。执行过程可能会报错：在Solaris上，如果错误消息指示系统无法找到共享库libjava.so(或JDK 1.2以上版本中的libjvm.so)，则需要将包含虚拟机库的目录添加到LD_LIBRARY_PATH变量中。在Windows上，错误消息可能表明它无法找到动态链接库javai.dll(或JDK 1.2以上版本中的jvm.dll)。在这种情况下，请将包含DLL的目录添加到系统环境变量中。

### 链接未知的JVM
如果本地程序需要连接多个未知的JVM，就不能指定某个特定的实现了JVM的本地库名(比如javai.dll, jvm.dll等)。一个解决办法是在运行时动态加载JVM库。下面的例子通过给定的路径加载`JNI_CreateJavaVM`的入口:
```c
/* Win32 version */ 
void *JNU_FindCreateJavaVM(char *vmlibpath) 
{    
    HINSTANCE hVM = LoadLibrary(vmlibpath);    
    if (hVM == NULL) {        
        return NULL;    
    }    
    return GetProcAddress(hVM, "JNI_CreateJavaVM"); 
}
```
`LoadLibrary`和`GetProcAddress`是Windows上动态链接的API，最好传递本地库的绝对路径给`JNI_FindCreateJavaCM`。<br>
Solaris版本类似:
```c
/* Solaris version */ 
void *JNU_FindCreateJavaVM(char *vmlibpath) 
{    
    void *libVM = dlopen(vmlibpath, RTLD_LAZY);    
    if (libVM == NULL) {        
        return NULL;    
    }    
    return dlsym(libVM, "JNI_CreateJavaVM"); 
}
```

## 附加本地线程
假设你有一个多线程应用，比如用C实现的web服务器，当多个HTTP请求进来时，服务器会创建多个线程来同步的处理这些HTTP请求。为了使得多个线程都能同时操作JVM，我们将JVM嵌入到服务器中，如下图所示。
![Embedding the Java virtual machine in a web server ]({{site.data.strings.blog_url}}JNI7-1.png)<br>

服务器创建的本地方法生命周期一般比JVM要短。因此，我们需要将本地线程附加到正在运行的JVM中去，然后在这个本地方法中执行JNI调用，最后在不影响其他线程的情况下与JVM分离。<br>

下面的例子attach.c说明了如何使用“调用接口”将本地线程附加到JVM中去，只适用于Windows平台。

```c
/* Note: This program only works on Win32 */ 
#include <windows.h> 
#include <jni.h> 
JavaVM *jvm; /* The virtual machine instance */
#define PATH_SEPARATOR ';' 
#define USER_CLASSPATH "." /* where Prog.class is */

void thread_fun(void *arg) 
{    
    jint res;    
    jclass cls;    
    jmethodID mid;    
    jstring jstr;    
    jclass stringClass;    
    jobjectArray args;    
    JNIEnv *env;    
    char buf[100];    
    int threadNum = (int)arg;    
    /* Pass NULL as the third argument */ 
#ifdef JNI_VERSION_1_2 
    res = (*jvm)->AttachCurrentThread(jvm, (void**)&env, NULL); 
#else    
    res = (*jvm)->AttachCurrentThread(jvm, &env, NULL);
#endif    
    if (res < 0) {       
        fprintf(stderr, "Attach failed\n");       
        return;    
    }    
    cls = (*env)->FindClass(env, "Prog");    
    if (cls == NULL) {        
        goto detach;    
    }    
    mid = (*env)->GetStaticMethodID(env, cls, "main", "([Ljava/lang/String;)V");    
    if (mid == NULL) {        
        goto detach;    
    }    
    sprintf(buf, " from Thread %d", threadNum);    
    jstr = (*env)->NewStringUTF(env, buf);    
    if (jstr == NULL) {        
        goto detach;    
    }    
    stringClass = (*env)->FindClass(env, "java/lang/String");    
    args = (*env)->NewObjectArray(env, 1, stringClass, jstr);    
    if (args == NULL) {        
        goto detach;    
    }
    (*env)->CallStaticVoidMethod(env, cls, mid, args);
 detach:    
    if ((*env)->ExceptionOccurred(env)) {        
        (*env)->ExceptionDescribe(env);    
    }    
    (*jvm)->DetachCurrentThread(jvm); 
}

main() {    
    JNIEnv *env;    
    int i;    
    jint res;
#ifdef JNI_VERSION_1_2    
    JavaVMInitArgs vm_args;    
    JavaVMOption options[1];    
    options[0].optionString =        
        "-Djava.class.path=" USER_CLASSPATH;
    vm_args.version = 0x00010002;    
    vm_args.options = options;    
    vm_args.nOptions = 1;    
    vm_args.ignoreUnrecognized = TRUE;    
    /* Create the Java VM */    
    res = JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args);
#else    
    JDK1_1InitArgs vm_args;    
    char classpath[1024];    
    vm_args.version = 0x00010001; 
    JNI_GetDefaultJavaVMInitArgs(&vm_args); 
    /* Append USER_CLASSPATH to the default system class path */    
    sprintf(classpath, "%s%c%s", vm_args.classpath, PATH_SEPARATOR, USER_CLASSPATH);    
    vm_args.classpath = classpath;    
    /* Create the Java VM */    
    res = JNI_CreateJavaVM(&jvm, &env, &vm_args); 
#endif /* JNI_VERSION_1_2 */
    if (res < 0) {        
        fprintf(stderr, "Can't create Java VM\n");
        exit(1);    
    }    
    for (i = 0; i < 5; i++)        
        /* We pass the thread number to every thread */
        _beginthread(thread_fun, 0, (void *)i);    
    Sleep(1000); /* wait for threads to start */    
    (*jvm)->DestroyJavaVM(jvm); 
}
```
本地代码开启了五个线程，线程开启后，会等待1s让线程运行完毕，完后调用`DestroyJavaVM`。每一个线程都将自己附加到JVM中，然后调用`Prog.main`，最后将自己与JVM分离。<br>
`JNI_AttachCurrentThread`的第三个参数为NULL，JDK 1.2以上版本引入`JNI_ThreadAttachArgs`这个结构体，它允许你添加额外的参数，如线程组等。这个结构体会在后面的文章中介绍。<br>
当程序执行`DetachCurrentThread`时，它会释放当前线程的所有局部引用。
以下是运行结果(顺序是随机的):
```
Hello World  from Thread 1
Hello World  from Thread 4
Hello World  from Thread 0
Hello World  from Thread 3
Hello World  from Thread 2
```