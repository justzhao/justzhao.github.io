---
layout: post
title: JAVA InterruptedException 
category: 技术基础
tags: JAVA InterruptedException

---

### 介绍

在Java 多线程中，使用sleep，wait 等方法的时候总是会抛出一个InterruptedException 异常，这个东西的作用到底是何解，我们应该如何处理。 其实字面意思就是中断的意思，我们可以理解为线程的退出。看下面Thread类中的方法

	public void stop(); //强行的退出线程，这种方法应该是要被抛弃的，
	因为一个线程正在执行当前的工作，这时候强行退出线程，合理吗？
    
    public void interrupt();//线程类中有一个标识位，表示当前线程是可中断的，设置为true。
    
	public static boolean interrupted();//返回当前的标志位，这是一个静态方法，但是此方法会清空标志位。


了解上面三个方法之后，我们在看下下面的常用的方法

	public static native void sleep(long millis) throws InterruptedException;
    //此方法是Thread中的，

    public final void wait() throws InterruptedException；//Object中的方法

    public final synchronized void join() throws InterruptedException；//等待线程

其实这些方法的内部都有这样的代码块

	if (Thread.interrupted()) {
      throw new InterruptedException(); //如果当前标志位为true就会爆出异常，
    }
  


   由此我们可以发现，一些能阻塞当前线程的方法时候，如果当前标志位为ture，会抛出 InterruptedException 异常，作用何在，个人理解是在于如果某个线程被阻塞了，我们也可以使用某种方式结束它。我们能指定一旦调用了这些阻塞方法，标志位会被清空。


看一个例子，如下代码：
	public class MyInterrupter {
	
	public static void main(String[] args) {
		 final Object o=new Object();
		
		Thread A=new Thread(new Runnable() {
			
			@Override
			public void run() {
				// TODO Auto-generated method stub
				
				synchronized (o) {
					try {
						o.wait();
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						
						System.out.println("我被打断了，可以退出了");
						//e.printStackTrace();
						System.out.println("xxx");
						return;
					}
						System.out.print("A");
					
						}
				}
					
		

		});
		
		A.start();
	
		A.interrupt(); //打断标志位设置为true ，它现在处于中断状态， 一旦他被阻塞就会抛出异常
	}
 	}

此代码的输出结果如下：

	我被打断了，可以退出了
	xxx

可以看到线程可以被正确的停止，而且没有输出A字符。如果我们不想在线程内部处理异常，可以把异常抛送出去，但是我们知道线程的标志位会被清空，所以更多的时候需要重新设置标志位，代码块如下。
	

	try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
				
		Thread.currentThread().interrupt(); // 重新设置
        throw new MyException(e);//抛出自定义的异常		
		}

这些我们在上层的代码中就可以处理这些异常。




### 理解

 个人理解，Java InterruptedException在多线程中主要在于优雅的告诉线程应该如何退出，以及遇到线程阻塞的清空下如何退出。
