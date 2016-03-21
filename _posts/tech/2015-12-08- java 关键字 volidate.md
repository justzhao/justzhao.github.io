---
layout: post
title: JAVA volatile 
category: 技术基础
tags: JAVA volatile 

---

### 介绍

volatile  是Java中的一个关键字，主要用来修饰某个变量。

个人理解，主要使用场景在多线程中，用来保证某变量在多线程能保证数据的一致性，但是它不不能保证操作上的原子性，网上很多时候叫它为弱synchronized。

举一个最通俗的例子，A线程修改变量buff后，B线程马上能知道变量buff已经被改变。

系统介绍volatile前，先介绍下java的内存模型：

Java内存模型主要是用来解决线程之间通信的模型。常说JMM。

JMM在JVM中定义了一个主内存的概念，并且规定所有的变量必须在主内存中生成，此外，每个线程都有自己的工作内存，工作内存中存放着，主内存中的变量的副本。

可以这样解释：

线程 A   有自己的工作内存 和主内存 M有通信。线程B有自己的工作内存，和主内存M有通信。


这样在一个变量在主内存中产生后，如果在多个线程使用，会产生数据的不一致性。所以我们就需要线程直接的同步。

JVM中规定，线程每次读取volatile 修饰的变量，都会先强制的从主内存中读取，在工作内存修改后的变量
都会强制刷回主内存，这样保证每次从主内存中读取的变量都是最新的数据。但是请注意:并没有保证操作的原子性。

例子，请看如下代码：
	
	public class JavaVolidate {
		public static void main(String[] args) throws InterruptedException {
	
		Worker w=new Worker();
		w.setDone(false);
		System.out.println("work start");
		w.start();
		 Thread.sleep(1000);
	    w.setDone(true);
		System.out.println("work stop");
	   	}
	}

	class Worker extends Thread{  
	   private volatile boolean done;  
	   public void setDone(boolean done){  
	      this.done = done;  
	   }  
	   @Override
	   public void run(){  
	      while(!done){  
	         System.out.println("working");  
	      }  
	      
	      System.out.println("worker exit");
	   }  
	}  


输出结果为：

	working start
	working
	working
	working
	working
	working
	working
	working
	working
	working
	working
	work stop
	worker exit

例子为主线程停止子线程的例子，发现worker exit 总是会在work stop后输出，也就是主线程操作变量done后,work线程会马上知道此变量发生变化，从而退出线程。


### 理解
需要注意的是，volatile 并不保证原子操作性，使用1000个线程对volatile变量做加1。最后的结果并不会等于1000。
