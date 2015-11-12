---
layout: post
title: JAVA CylicBarrier
category: 技术
tags: JAVA CylicBarrier

---

### 介绍

CylicBarrier  个人理解就是多个线程的栅栏，并且可以循环使用。还是举老师带学生去公园玩的例子，老师宣布上午大家自由玩耍，中午到西门集合。集合之后下午大家在自由玩耍，最后到南门集合，离开公园。
这里提醒出一组线程多次协同工作，共同完成一个任务。



CylicBarrier 初始化的时候需要一个内部计算器，我们可以理解为共同工作的线程数目，也就是需要到达一个栅栏的线程的个数。
	
	CylicBarrier(int n);//n是一个整数，代表计数器的个数
    CylicBarrier(int n,Runnable);//表示当所有线程到达栅栏的时候可以先执行Runnable线程，提供更丰富的功能     
    void await();//在线程中调用，表示当前线程已经到达栅栏。

使用CylicBarrier可以用于多线程计算，最后合并各个计数的结果等，下面还是一个老师带学生游玩的例子

	import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class ParkStudentCyclicBarrier {
	
	
	 private static boolean over = false;
	
	public static void main(String[] args) {
		CyclicBarrier cb=new CyclicBarrier(10, new Runnable() {
			
			@Override
			public void run() {
				// TODO Auto-generated method stub
				
			
				System.out.println("所有同学已经集合完毕，准备下一步活动");
				
			}
		});
		
		System.out.println("上午开始游玩");
		for(int i=1;i<=10;i++){
			new MyStudent(i, cb).start();
		}

	
	}

	}

	class MyStudent extends Thread{
	
	private int num;
	private CyclicBarrier cb;
	
	public MyStudent(int num, CyclicBarrier cb) {
		super();
		this.num = num;
		this.cb = cb;
	}

	@Override
	public void run() {
		
		try {
			
	
				System.out.println("第"+num+"个学生上午自由玩耍完毕，回到集合点");
		
			
			cb.await();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (BrokenBarrierException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

		
		try {
			
		
				System.out.println("第"+num+"个学生下午自由玩耍完毕，回到集合点 ");
			
			
			cb.await();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (BrokenBarrierException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

	}
	}

 整个代码打印结果如下：

	上午开始游玩
	第1个学生上午自由玩耍完毕，回到集合点
	第2个学生上午自由玩耍完毕，回到集合点
	第6个学生上午自由玩耍完毕，回到集合点
	第3个学生上午自由玩耍完毕，回到集合点
	第4个学生上午自由玩耍完毕，回到集合点
	第7个学生上午自由玩耍完毕，回到集合点
	第5个学生上午自由玩耍完毕，回到集合点
	第8个学生上午自由玩耍完毕，回到集合点
	第10个学生上午自由玩耍完毕，回到集合点
	第9个学生上午自由玩耍完毕，回到集合点
	所有同学已经集合完毕，准备下一步活动
	第9个学生下午自由玩耍完毕，回到集合点 
	第4个学生下午自由玩耍完毕，回到集合点 
	第3个学生下午自由玩耍完毕，回到集合点 
	第6个学生下午自由玩耍完毕，回到集合点 
	第2个学生下午自由玩耍完毕，回到集合点 
	第5个学生下午自由玩耍完毕，回到集合点 
	第8个学生下午自由玩耍完毕，回到集合点 
	第1个学生下午自由玩耍完毕，回到集合点 
	第10个学生下午自由玩耍完毕，回到集合点 
	第7个学生下午自由玩耍完毕，回到集合点 
	所有同学已经集合完毕，准备下一步活动


  使用Cyclicbarrier可以比CountDownLatch有更丰富的属性，还包括如下一些方法

	void reset();//重置计算器
    int getNumWaiting();//获取当前被栅栏阻塞的线程数目




 
### 理解

CountDownLatch和CycLicBarrier都是Java并发包中的重要的同步工具类，类似但是又有区别，其中CycLicBarrier有更丰富的语意，适用于更复杂的业务场景，共同求和等。


	



