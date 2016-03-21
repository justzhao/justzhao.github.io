---
layout: post
title: JAVA CountDownLatch
category: 技术基础
tags: JAVA CountDownLatch

---

### 介绍

CountDownLatch  个人理解就是一个线程等待其他线程完成任务，最通俗的例子就是
老师带着一群学生去公园玩，老师宣布大家自由玩耍，玩耍后一定要在公园集合，一定要等到大家都集合完毕才有下一步的工作，使用技术语言就是，允许一个进程等待其他线程来完成工作。

CountDownLatch 初始化的时候需要一个内部计算器，我们可以理解为共同工作的线程数目，这个类一些常用的三个方法如下:
	
	New CountDownLatch(n);//n是一个整数，代表计数器的个数
    void countDown(); //计算器减1
    void await();//等待计算器归0
		
		
使用此方法可以使多个线程共同协作。
通俗的来讲，就是CountDownLatch的构造函数接收一个int类型的参数作为一个计数器，如果想等待N个线程，就传入N个。然后调用 await()方法可以阻塞当前线程，直到CountDownLatch的计算器归0.

值得注意的是，CountDownLatch不可能重新初始化或者使用其他方法修改CountDownLatch对象的内部计算器的值。


下面代码是老师带学生去公园玩的例子。
	
	public class ParkStudent {
	
	   static CountDownLatch cd = new CountDownLatch(10); //一个老师带10个学生去玩
	   
	   
	   public static void main(String[] args) throws InterruptedException {
		   
		   System.out.println("开始到达公园");
		   System.out.println("公园各自游玩");
		   for( int i=0;i<10;i++){
			   
			 new Student(i,cd).start();
		   }
		   
		   cd.await(); //等待所有学生集合
		   System.out.println("集合完毕");
		
	}
	

	}


	class Student extends Thread{
	
	public int num;
	
	public CountDownLatch c;

	public Student(int num, CountDownLatch c) {
		super();
		this.num = num;
		this.c = c;
	}
	
	@Override
	public void run() {
		  System.out.println("学生 "+num+"开始自由活动");
	
		  c.countDown();
		  System.out.println("学生 "+num+"结束自由活动，集合完毕");
	}
	}

上面代码执行之后，主线程都会等待其他线程执行完毕，最后打印集合完毕。


 
### 理解

 使用CountDownLatch可以让多个线程共同合作，举个例子，可以让多个线程去处理一个相对大的文本，共同完成任务。这其实和join()方法类似，但是比join()更优秀，join()其实是不断轮询当前某个线程是否还存活，部分代码片段如下：
	
	while(isAlive()){
      wait(0);//持续等待下去。
	}



	



