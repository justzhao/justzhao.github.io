---
layout: post
title:  AQS的waitStatus
category: 技术基础
tags:AQS，线程
description:  AQS的waitStatus 状态变更
---
 
 
 #### AQS waitStatus 状态变迁  
 
 
 AQS 的waitStatus状态，表示当前Node 线程节点的状态，ws主要有以下几个值。wa初始值为0，一般使用cas来操作赋值。
    
        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        // 表示当前节点的后继节点需要唤醒
        static final int SIGNAL    = -1;
    
        /** waitStatus value to indicate thread is waiting on condition */
        // 表示阻塞在条件队列上
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
         // 一次releaseShared 方法会扩散到后继节点
        static final int PROPAGATE = -3;
        
        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         */    
    
    
#### CountDownLatch     
    
下面以CountDownLatch为例子。    

      class Driver { // ...
    void main() throws InterruptedException {
      CountDownLatch startSignal = new CountDownLatch(1);
      CountDownLatch doneSignal = new CountDownLatch(N);
 
      for (int i = 0; i < N; ++i) // create and start threads
        new Thread(new Worker(startSignal, doneSignal)).start();
 
      doSomethingElse();            // don't let run yet
      startSignal.countDown();      // let all threads proceed
      doSomethingElse();
      doneSignal.await();           // wait for all to finish
    }
    }
 
      class Worker implements Runnable {
        private final CountDownLatch startSignal;
        private final CountDownLatch doneSignal;
        Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
           this.startSignal = startSignal;
           this.doneSignal = doneSignal;
        }
        public void run() {
           try {
             startSignal.await();
             doWork();
             doneSignal.countDown();
           } catch (InterruptedException ex) {} // return;
        }
     
        void doWork() { ... }
      }
    
    
 上面是CountDownLatch的用法，下面主要介绍await和CountDown和waitStatus的状态    



    
#### await   

    
    // 等待count=0，当count=0的时候所有阻塞在countDownLatch的线程都会启动。
    
    
     public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }



    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
            // 如果当前计数器大于0，则返回-1，表示当前条件没有接续。，需要入队。
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

     
       
    // 如果当前count 计数器=0，表示条件已经就绪,返回>0
    protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
    }

    

    // 共享锁的入队操作。
      private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
       // 队列尾部添加一个节点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        
        try {
        
            for (;;) {
               // 如果当前节点的前驱是头节点，
                final Node p = node.predecessor();
                if (p == head) {
                // 头节点是需要唤醒后继节点的。
                
                // 再次尝试获取资源，
                // 当count=0的时候，表示计数器=0，所有条件已经就绪.countDownLatch这里，r=1，此时，把当前节点设置为头节点。唤醒后继节点。
                // 否则一直for循环。
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                // 如果当前节点的前驱不是头节点，且设置前驱的waitStatus=singal，表示前驱需要唤醒当前节点。并且阻塞自己
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    
    
    
     // 获取资源失败后，是否要阻塞当前线程
     // 根据前驱节点来判断是否要阻塞当前线程。
      private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
        //ws=-3 或者ws=0 设置前驱节点的ws=singnal ，表示需要前驱节点唤醒。 
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

    
    // 表示当前节点已经获取到资源，  设置新的头节点为当前节点，这里 propagate=1
     private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
         1>0  表示还有资源可用，不需要阻塞了。
        if (propagate > 0 || h == null || h.waitStatus < 0) {
            Node s = node.next;
            // 唤醒下一个节点。
            if (s == null || s.isShared())
                // 让头节点开始唤醒
                doReleaseShared();
        }
    }
    
    
     //唤醒，如果头节点的状态是siganl 表示需要唤醒。
     //如果头节点的状态=0,则设置为PROPAGATE
     private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {// for 循环会唤醒所有的节点。
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                // 如果当前节点为ws为初始化状态，则设置为-3.传播性。
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
    
    
    
    


##### countDown 方法

    // 让计数器-1
    public void countDown() {
        sync.releaseShared(1);
    }
    
    
    //
      public final boolean releaseShared(int arg) {
      //计数器=0的时候返回ture，这时候可以唤醒所有的线程
        if (tryReleaseShared(arg)) {
            //执行唤醒。
            doReleaseShared();
            return true;
        }
        return false;
    }


     protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                    return nextc == 0;
        }
    }


    
    // 唤醒所有阻塞在共享锁上的线程
      private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            // 获取头节点，用头节点来唤醒后继。初始化的共享锁的头节点是一个空Node
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // 如果当前节点ws=SIGNAL 表示需要唤醒后继节点。
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                        // 唤醒后继节点
                    unparkSuccessor(h);
                }         //如果ws =0 表示无后继节点，设置ws=-3
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }


当某个节点获取凭证(获取锁)的情况下，需要唤醒后继节点。如果当前节点无后继节点，或者说后继节点还没加入到同步队列，会先把当前节点ws=PROPAGATE。当后继节点进入队列后，会修改前驱节点PROPAGATE为SIGNAL。这样前驱节点就可以唤醒后继节点了。






       private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        // ws<0的情况有SIGNAL ，CONDITION，PROPAGATE，然后设置为0.
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
    

#### waitStatus状态


线程 A await之后，进入队列。设置它的前驱 也就是head节点ws=SINGAL

线程 B await之后，进入队列。设置它的前驱 也就是A节点ws=SINGAL

线程 C await之后，进入队列。设置它的前驱 也就是B节点ws=SINGAL

此时队列结构如下图。

![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-28/70897961.jpg)


线程D 调用CountDown之后，计数器count=0,开始执行doReleaseShared 方法，依次唤醒abc节点。

当唤醒a节点的线程之后，a节点发现它的前驱是头节点，就会设置自己为头节点，然后继续唤醒a节点的后继。

![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-29/29403116.jpg)


