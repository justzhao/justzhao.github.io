---
layout: post
title: Java Synchronized 原理
category: 技术基础
tags: Java Synchronized 同步
description:  Synchronized 
---

#### Synchronized

一种对象内置锁，用来线程同步。

##### 作用对象

1. 代码块
    修饰代码的时候，可以使用object的锁和实例的锁。synchronized锁住的是对象实例还是类方法。


	public class SynchronizedTest {
	    
	    public void sendImage(){
	        //锁住的是本类，全局锁，所以在任何时候都只会有一个线程访问同步块
	        synchronized (SynchronizedTest.class) {
	            
	        }
	    }
	    
	    public void sendVideo(){
	        //锁住的是本实例对象，如果有三个实例对象，就会有三个线程访问同步块
	        synchronized (this) {
	            
	        }
	    }
	
	}


   
2. 普通方法


	    //普通方法，锁住的是本实例对象，一个对象任何时候只能一个线程访问synchronized方法
	    public synchronized void sendMessage(){
	        
	        System.out.println("hello!");
	    }



3. 静态方法


		// 静态方法，锁住的是本类，全局锁。任何时候只有一个线程访问本类的方法

	    public synchronized  static void sendRadio(){
	        
	        System.out.println("hello!");
	    }



##### 代码块实现原理


###### monitorenter

每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

- 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。

- 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.

- 3.如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权

###### monitorexit
- 1执行monitorexit的线程必须是objectref所对应的monitor的所有者。

- 指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

##### 方法实现原理

其常量池中多了ACC_SYNCHRONIZED标示符
JVM就是根据该标示符来实现方法的同步的：当方调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。





##### synchronized 实现单例模式



	
	public class SingletonClass {
	    private static SingletonClass instance;
	
	    private SingletonClass() {
	
	    }
	
	    public synchronized static SingletonClass getInstance() {
	        if (instance == null) {
	            instance = new SingletonClass();
	        }
	
	        return instance;
	    }
	
	    //双重加锁
	    public static SingletonClass getInstanceByDouble() {
	        if (instance == null) {
	            synchronized (SingletonClass.class) {
	                if (instance == null) {
	                    instance = new SingletonClass();
	                }
	            }
	        }
	        return instance;
	    }
	
	    //静态内部类
	    private static class SingletonHolder {
	        private static final SingletonClass INSTANCE = new SingletonClass();
	    }
	
	    public static SingletonClass getInstanceByStatic() {
	
	        return SingletonHolder.INSTANCE;
	    }
	
	}
	
	//此外枚举实现单例模式
	
	public enum Singleton {
	
	    INSTANCE;
	    public void whateverMethod() {
	    }
	}


#### Synchronized 在JVM 中的实现

根据前面所知，Synchronized 是通过任何一个对象中的monitor 来实现的。

monitor 会操作操作系统的内核来实现同步机制，操作系统可能从内核态到用户态之间的转化，是非常消耗资源的。在JDK1.6 以后，JVM 对Synchronized 做了许多的优化，引出了 重量级锁，轻量级锁，偏向锁的概念。

HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

在jvm中，monitor是存在于对象头中。 Hotspot虚拟机的对象头主要包括两部分数据：Mark Word（标记字段）、Klass Pointer（类型指针）。

- Klass Point是是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例，

- Mark Word用于存储对象自身的运行时数据，它用来实现轻量级锁和偏向锁。

##### Mark Word

Mark Word存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。
Java对象头一般占有两个机器码（在32位虚拟机中，1个机器码等于4字节，也就是32bit）下图是Java对象头的存储结构（32位虚拟机），其中Mark Word 占用一个机器码。
初始无锁状态如下

| 25bit | 4bit | 1bit | 2bit
| ---- | ---- | ---- |----|
|对象的hashCode | 对象的分代年龄 | 是否偏向锁 | 锁的标志位
|对象的hashCode | 对象的分代年龄 | 0| 01

锁的状态总共有四种：无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁（但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级）,
状态的变化如下所示:

![image](http://7x00ae.com1.z0.glb.clouddn.com/markword-info.jpg)



##### 偏向锁

######获取锁

偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令。

1. 访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01——确认为可偏向状态。
2. 如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤5，否则进入步骤3
3. 如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行5；如果竞争失败，执行4。
4. 如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。
5. 进入同步代码。




###### 释放锁

偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状


##### 轻量级锁

######获取锁


1. 在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。

2. 拷贝对象头中的Mark Word复制到锁记录中。

3. 拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向对象的 mark word。如果更新成功，则执行步骤3，否则执行步骤4。

4. 如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态。

5. 如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。


######释放锁


1. 通过CAS操作尝试把线程中复制的Displaced Mark Word对象替换当前的Mark Word。

2. 如果替换成功，整个同步过程就完成了。

3. 如果替换失败，说明有其他线程尝试过获取该锁，那就要在释放锁的同时，唤醒被挂起的线程。

#### 总结

本文介绍了Synchronized 使用方法，和底层实现原理，并且了解了Java 对象头的相关知识，偏向锁，轻量级锁等知识。


##### 参考资料

java 并发编程的艺术

http://www.infoq.com/cn/articles/java-se-16-synchronized