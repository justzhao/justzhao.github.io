---
layout: post
title: Java Object
category: 技术基础
tags:  Java Object
description: Java Object 基本对象
---
#### 概念


在java中，Object是对象的意思，Java中任何对象都默认继承了Object类，
所以Object类是任何类的父类，可以使用Object 执行任何对象实例。

```
 Object ob=new Pojo();
 
```

Object 类中某个的构造方法

```
public Object(){}
```
在任何类实例化的时候，都会先调用这个构造方法。


Java 对象的堆内存占用由以下组成：

- 一个对象头信息，由几字节的基本元信息组(在 32-bit JVM 上占用 64bit， 在 64-bit JVM 上占用 128bit 即 16 bytes)
- 原始类型字段的内存占用由它们的大小决定（比如int 占用4个字节）
- 引用字段的内存占用（每个 4 字节，对象属性）
- 对齐：为了让每个对象的开始地址是字节的整数倍、减少表示对象指针需要的比特数，对象数据后面会添加一些“没用”的数据，来实现对齐。


#### 源码

    public class Object {
    
    // 调用静态native方法
    private static native void registerNatives();
    static {
        registerNatives();
    }

    
    public final native Class<?> getClass();

    
     // 类的hashcode，用来标记一个类的。常用与equals方法
    public native int hashCode();

    // 两个对象是否相等，需要被子类覆盖，自定义两个对象相等的条件
    public boolean equals(Object obj) {
        return (this == obj);
    }

    // 类熟悉的类需要实现 Cloneable接口
    protected native Object clone() throws CloneNotSupportedException;

    
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }

    /** 下面是一些线程同步的方法 */
    
    public final native void notify();

    
    public final native void notifyAll();

    
    public final native void wait(long timeout) throws InterruptedException;

    
    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && timeout == 0)) {
            timeout++;
        }

        wait(timeout);
    }

    
    public final void wait() throws InterruptedException {
        wait(0);
    }

    // 当次对象要被垃圾回收器回收的时候，会调用此方法。
    protected void finalize() throws Throwable { }
}


#### 解析

#### equals(Object ob)

指示某个其他对象是否与此对象“相等”。

equals 方法在非空对象引用上实现相等关系：

自反性：对于任何非空引用值 x，x.equals(x) 都应返回 true。

对称性：对于任何非空引用值 x 和 y，当且仅当 y.equals(x) 返回 true 时，x.equals(y) 才应返回 true。

传递性：对于任何非空引用值 x、y 和 z，如果 x.equals(y) 返回 true，并且 y.equals(z) 返回 true，那么 x.equals(z) 应返回 true。

一致性：对于任何非空引用值 x 和 y，多次调用 x.equals(y) 始终返回 true 或始终返回 false，前提是对象上 equals 比较中所用的信息没有被修改。
对于任何非空引用值 x，x.equals(null) 都应返回 false。

Object 类的 equals 方法实现对象上差别可能性最大的相等关系；即，对于任何非空引用值 x 和 y，当且仅当 x 和 y 引用同一个对象时，此方法才返回 true（x == y 具有值 true）。

注意：当此方法被重写时，通常有必要重写 hashCode 方法，以维护 hashCode 方法的常规协定，该协定声明相等对象必须具有相等的哈希码



#### 对象的拷贝 

    protected Object clone()
    
对象的clone。需要配合Cloneable接口使用。如果不继承则会抛出CloneNotSupportedException 异常。一般而言是浅拷贝：（拷贝基本成员属性，对于引用类型仅返回指向改地址的引用）

当一个对象的属性还有复杂对象的时候，对象属性的类也需要实现Cloneable 接口，才能实现深拷贝。


    public Object clone() throws CloneNotSupportedException{
    
        return super.clone();
    }

##### 深拷贝：

    public class MyObject implements Cloneable {
   
    public int num = 0;
    public String str;
    public A a;
 
    public Object clone() throws CloneNotSupportedException {
        MyObject o = (MyObject) super.clone();
        o.str = new String(this.str);
        o.a = (A) a.clone();
        return o;
    }
}
 
    // 成员属性A必须为Cloneable的
    class A implements Cloneable {
    public Object clone() throws CloneNotSupportedException {
            return super.clone();
        }
    }

