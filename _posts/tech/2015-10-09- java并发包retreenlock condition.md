---
layout: post
title: java面试题 三个线程打印ABC
category: 技术
tags: JAVA retreenlock condition

---

### 介绍

retreenlock 就是锁，能和synchronize 一样的同步效果，但是retreenlock有更多的特性。
condition 个人可以理解为条件变量，通知机制。

ReentrantLock 类实现了 Lock 类，允许把锁定的实现作为 Java 类
它一般需要如下模型

	Lock lock = new ReentrantLock();  
    lock.lock();  
    try {   
      // update object state  
    }  
    finally {  
      lock.unlock();   
    }  


retreenlock 更多特性:

一般来说，retreenlock是非公平锁，公平锁使线程按照请求锁的顺序依次获得锁；而不公平锁则允许线程有时可以比先请求锁的其他线程先得到锁，这样在实际应用中会更有意义。

当某个线程等待 retreenlock时间过长的时候，可能会自动放弃等待锁。
 
### 理解
使用lock()和unlock()修饰的代码块是原子性的，这里和synchronize的效果一样，
但是unlock一定要在try catch finnaly的finally中释放，当try中的代码块出问题的时候，就不会释放当前的对象锁。
先说java 面试题，三个线程答应ABC的情况，使用retreenlock condition如何实现，先看代码

	public class Main {
	private static int count = 0; //计算器
	
	public static void main(String[] args) throws InterruptedException {
		final Lock loc = new ReentrantLock(); //java并发包中的锁
		final Condition con=loc.newCondition(); //我理解的条件变量

		Thread A=new Thread(new Runnable() {
			
			@Override
			public void run() {
				while(count<30){
					try {
						loc.lock();
						
						//loc.tryLock();
						if(count%3==0){
							System.out.print("A");
							count++;
							
							con.signalAll();
						}else{
							con.await();
						}
						
					} catch (Exception e) {
						// TODO: handle exception
					}finally{
						loc.unlock();
					}
				
					
					
				}
				
			}
		});
		
		
		Thread B=new Thread(new Runnable() {
			
			@Override
			public void run() {

				while(count<30){
					
					
					try {
						loc.lock();
						if(count%3==1){
							System.out.print("B");
							count++;
							con.signalAll();
						}else{
							con.await();
						}
						
					} catch (Exception e) {
						// TODO: handle exception
					}finally{
						loc.unlock();
					}
				
					
					
					
					
				}
				
			
				
			}
		});
		
		
		Thread C=new Thread(new Runnable() {
			
			@Override
			public void run() {

				while(count<30){
					try {
						loc.lock();
						if(count%3==2){
							System.out.print("C");
							count++;
							con.signalAll();
						}else{
							con.await();
						}
						
					} catch (Exception e) {
						// TODO: handle exception
					}finally{
						loc.unlock();
					}
				
					
					
				}
				
			
				
			}
		});
		A.start();
		B.start();
		C.start();
		
		A.join();
		B.join();
		C.join();
		
		System.out.println(count);
	}

	}


上述代码中，使用了如下代码
	
	final Condition con=loc.newCondition();
Condition 我习惯把它称作条件变量，或者说通知变量，它和object对应的wait(),notify(),notifyAll()类似。但是也有更多的特性。
con.await()  让当前线程阻塞在con上，con.signalAll()。所以我们在实际应用中可以创建多个Condition来操作不同的线程，比如说生产者和消费者。
以下代码取自arrayblockQueue中的put和get方法。
	

		  
        private final Condition notEmpty;//用来控制消费者的条件变量

        private final Condition notFull;//用来控制生产者的条件变量

        //put 方法使用 条件变量，当队列满了之后使用notFull阻塞当前线程
	    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        final E[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            try {
                while (count == items.length)
                    notFull.await();
            } catch (InterruptedException ie) {
                notFull.signal(); // propagate to non-interrupted thread
                throw ie;
            }
            insert(e);
        } finally {
            lock.unlock();
        }
    }
	

    //take 方法  当变量空了之后使用notEmpty来阻塞当前线程
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            try {
                while (count == 0)
                    notEmpty.await();
            } catch (InterruptedException ie) {
                notEmpty.signal(); // propagate to non-interrupted thread
                throw ie;
            }
            E x = extract();
            return x;
        } finally {
            lock.unlock();
        }
    }

	



	



