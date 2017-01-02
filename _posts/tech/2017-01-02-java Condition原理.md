---
layout: post
title: Java Condition原理
category: 技术基础
tags: Java Condition 原理
description:  Condition 源码
---

### Condition

Condition 称为 条件变量，Condition 将 Object 监视器方法（wait、notify 和 notifyAll）改进， 配合Lock 对象使用，实现多线程之间的同步。先看如下方法。自jdk 1.5被引入，主要是用于多线程直接同步操作资源。

### 解析

Condition接口如下

	public interface Condition {

   
    //阻塞当前线程直到其他线程唤醒他。
    void await() throws InterruptedException;

    //阻塞当前线程直到其他线程唤醒他，不抛出InterruptedException
    void awaitUninterruptibly();

    //阻塞当前线程直到其他线程唤醒他， 参数nanosTimeout 是最长等待时间，单位微秒，
    //返回值是nanostimeout 值减去花费在等待此方法的返回结果的时间，小于0表示没有时间。
    long awaitNanos(long nanosTimeout) throws InterruptedException;

     //阻塞当前线程直到其他线程唤醒他，并且设置等待时间
    boolean await(long time, TimeUnit unit) throws InterruptedException;

    
    boolean awaitUntil(Date deadline) throws InterruptedException;

    // 唤醒某个线程，会重新竞争锁
    void signal();

    // 唤醒所有的线程，会重新竞争锁
    void signalAll();



Condition 实例化主要是由他的子类ConditionObject 。它是AQS的一个内部类。
以下是ConditionObject 的实现。

	 public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
  
        
        // 等待队列节点队首
        private transient Node firstWaiter;
     
        
        // 等待队列节点队尾
        private transient Node lastWaiter;

        
        public ConditionObject() { }

        // Internal methods

        
        // 在队列尾部加入一个等待节点。
        
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // 如果尾部节点是取消状态则删除，
            if (t != null && t.waitStatus != Node.CONDITION) {
            
                // 删除队尾节点
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }

      //signal就是唤醒Condition队列中的第一个非CANCELLED节点线程
      //transferForSignal 调用的是AQS的方法
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&(first = firstWaiter) != null);
        }


        //而在transferForSignal中，如果节点的waitStatus不是CONDITION，
        //那么就只会是CANCELLED（在await操作中执行fullyRelease时，如果失败会将节点的waitStatus设置到CANCELLED）
        final boolean transferForSignal(Node node) {
       
         // 如果不能改变他的waitStatus状态为0.就证明当前状态为canceled
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
       
         // 成功取消 Condition状态，状态改为0.就进入AQS队列。
         // 获取当前node节点的前驱p
        Node p = enq(node);
        int ws = p.waitStatus;
        
        // 如果前驱节点的状态为cancel 或者修改waitStatus失败，则直接唤醒
        // sinal状态表示当前节点的后驱被阻塞了。
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
        
       // 唤醒所有
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }

       
         
         //从等待队列 删除所有的等待的节点，
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }

        // public methods

       
         // 取Condition队头，做唤醒操作。
         // 判断当前线程是否持有锁，如果没有持有，则抛出异
        public final void signal() {
            
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        // 唤醒Condition队列中的每个节点。
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }

      
        public final void awaitUninterruptibly() {
            Node node = addConditionWaiter();
            long savedState = fullyRelease(node);
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if (Thread.interrupted())
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted)
                selfInterrupt();
        }

       

        /** Mode meaning to reinterrupt on exit from wait */
        private static final int REINTERRUPT =  1;
        /** Mode meaning to throw InterruptedException on exit from wait */
        private static final int THROW_IE    = -1;

        
         // 在唤醒之前如果被中断了就返回-1，如果被唤醒了返回1，没有中断返回0.
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ? (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :  0;
        }

      
         // 在阻塞后检测状态。
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            // 抛出异常
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
                //中断自己
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

        //实现await 方法       
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            // 这里当前线程释放锁，并且唤醒后继节点，后继节点把自己设置为头节点，并且删除当前节点
            long savedState = fullyRelease(node);
            int interruptMode = 0;
             //判断当前线程是是否在AQS队列
             //如果不在AQS等待队列中，就park当前线程，如果在，就退出循环，这个时候如果被中断，那么就退出循环
             //不在AQS中就阻塞此线程，等待其他线程唤醒。
             
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // acquireQueued 线程被唤醒之后会再次尝试获取锁，
            // 如果获取不到锁会再次阻塞。
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
        
        // 实现await 方法
        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            if (unit == null)
                throw new NullPointerException();
            
            long nanosTimeout = unit.toNanos(time);
            
            if (Thread.interrupted())
                throw new InterruptedException();
            // 在condition队尾新增一个节点，
            Node node = addConditionWaiter();
            // 释放当前的锁。
            long savedState = fullyRelease(node);
            long lastTime = System.nanoTime();
            boolean timedout = false;
            int interruptMode = 0;
            //判断当前线程是是否在AQS队列
            while (!isOnSyncQueue(node)) {
                // 如果当前等待时间小于0，马上取消等待状态
                if (nanosTimeout <= 0L) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                // 等待时间大于自旋等待的时间
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                    //如果被中断就退出循环。
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
            }
            // 此时 线程已经被唤醒，再次尝试获取锁。
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

       // await 的另外一种实现
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            long savedState = fullyRelease(node);
            long lastTime = System.nanoTime();
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;

                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return nanosTimeout - (System.nanoTime() - lastTime);
        }

       
       // await 的另外一种实现
        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            if (deadline == null)
                throw new NullPointerException();
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            long savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        

        //  support for instrumentation

        // 返回当前线程是否是Condition拥有着。
        final boolean isOwnedBy(AbstractQueuedLongSynchronizer sync) {
            return sync == AbstractQueuedLongSynchronizer.this;
        }

        
        // 是否有线程在等待
        protected final boolean hasWaiters() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    return true;
            }
            return false;
        }

        
        // 返回一个等待线程个数估计值
        protected final int getWaitQueueLength() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int n = 0;
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    ++n;
            }
            return n;
        }

       // 返回所有的等待节点
        protected final Collection<Thread> getWaitingThreads() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            ArrayList<Thread> list = new ArrayList<Thread>();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION) {
                    Thread t = w.thread;
                    if (t != null)
                        list.add(t);
                }
            }
            return list;
        }
    }

### 原理
通过源码我们可以看到ConditionObject 内部也维护了一个队列，我们叫做Condition队列，用来维护阻塞在Condition的线程。也就是在Lock内部其实维护了两个队列，另外一个是AQS队列。
在lock操作之后，如果线程被阻塞了就会被加入AQS队列尾部，如果没有被lock阻塞，在运行到await操作时候被阻塞就会加入到Condition队列中并且释放锁，当对应节点被signal唤醒后，会把节点从Condition对象中取出，加入到AQS队列中。
如下示意图

![](http://7x00ae.com1.z0.glb.clouddn.com/Condition%E9%98%9F%E5%88%97%E5%9B%BE.png)


### 总结
Condition线程同步操作主要还是通过AQS队列和Condition队列，Condition队列维护着等会信号的线程节点。所以要深刻理解Condition还是其他线程同步操作，都需要认真理解AQS队列。




