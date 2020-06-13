---
title: Java的内部类
# image: /assets/img/blog/...
description: >
  面试时被问到，为什么Java要支持内部类，当时懵懵懂懂说可以扩展Java的单继承机制...
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [Java]
---

Java允许在 一个类内部定义另外的一个类，这就是内部类。内部类分为两类：静态(static nested classes)和非静态(inner classes)。其中非静态的可以访问外部类的成员而静态的不可以(这里指的是外部类的非静态成员)。此外，内部类支持所有的权限修饰符。<br>

## **为什么要使用内部类**
* **对仅在一个地方使用的类进行逻辑分组：**如果一个类只在一个地方被使用，那么将其定义在该类的内部，将使得整个程序包更加简化
* **增加封装性：**考虑有两个类A和B，其中B需要访问A的成员，那么A的成员就不能定义为`private`，如果B为内部类，就可以访问A的私有成员。另外，内部类本身就可以被外部类隐藏(比如，对同一个包内的其他类不可见)。
* **增加可读性：**嵌套功能较少的内部类，可以增加代码的可读性。
* **多继承：**多定义几个内部类分别继承不同的父类，功能上这个外部类就具有多个父类的特征。

### **内部类是如何访问外部类成员的**
实际上是通过外部类的`this`来访问的。
```java
public class OuterClass {
    private int out = 1;
    class InnerClass{
        public void print(){
            System.out.println(out);
        }
    }

    public static void main(String[] args) {
        
    }
}
```
查看一下生成的.class文件，可以发现：
```java 
    class InnerClass {
        InnerClass() {
        }

        public void print() {
            System.out.println(OuterClass.this.out);
        }
    }
```

> 扩展一下：**Shadow**<br>
> 不同作用域的变量重名，更小的scope会隐藏掉更大的scope

```java 
public class ShadowTest {

    public int x = 0;

    class FirstLevel {

        public int x = 1;

        void methodInFirstLevel(int x) {
            System.out.println("x = " + x);
            System.out.println("this.x = " + this.x);
            System.out.println("ShadowTest.this.x = " + ShadowTest.this.x);
        }
    }

    public static void main(String... args) {
        ShadowTest st = new ShadowTest();
        ShadowTest.FirstLevel fl = st.new FirstLevel();
        fl.methodInFirstLevel(23);
    }
}
```

### **静态内部类**

> A static nested class interacts with the instance members of its outer class (and other classes) just like any other top-level class. In effect, a static nested class is **behaviorally a top-level class** that has been nested in another top-level class for packaging convenience.

### **非静态内部类**
**局部内部类：**
* 不允许使用权限修饰符
* 方法外不可见
* 可以引用局部变量，但该局部变量必须为`final`(JDK 1.8之后编译器会自动声明为`final`)

**匿名内部类：**
* 没有访问修饰符
* 必须继承一个类或者实现一个接口
* 没有静态成员或方法
* 没有构造方法，因为没有名字
* 可以引用局部变量，但必须声明为final

> 为什么引用的局部变量必须声明为`final`？**主要原因是内部类的生命周期要比局部变量的生命周期长，类对象在堆而局部变量在栈**


## **使用内部类的一些问题**

### **序列化**
> Serialization of inner classes, including local and anonymous classes, is strongly discouraged. 

非静态内部类默认会持有外部类的引用(*可以参考前文的`this`*)，如果外部类不可序列化，那么内部类的序列化肯定会报错的；而静态内部类就不会有这个问题，*可以参考前文引用的一段关于静态内部类的描述*。

### **内存泄漏**
[memory leaks in inner classes](https://www.javaworld.com/article/3526554/avoid-memory-leaks-in-inner-classes.html)，这种类型的内存泄漏原因就是非静态内部类持有一个外部类的引用，如果内部类被外部类以外的对象持有，那么该外部类是没法被JVM垃圾回收的(直到内部类被回收)。另外，如果外部类还申请了很大的内存，那么就很容易发生内存溢出错误了。