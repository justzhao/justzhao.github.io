---
layout: post
title:  Reference和ReferenceQueue
category: 技术基础
tags: 引用，引用队列
description:  Reference和ReferenceQueue
---

#### 引用对象 Reference


java 的四种引用入下图所示。

![image](http://7x00ae.com1.z0.glb.clouddn.com/image_20170109215552.png)

Final，Soft,Weak，Phantom都继承Reference类。

FinalReference是包权限的，意味着外面的类无法访问，不能被公开使用。

Finalizer 是FinalReference的一个实现。

Java 中 Object类是所有类的父类在Object类中有一个finalize方法

    protected void finalize() throws Throwable { }

如果某个对象子类覆盖了finalize的方法，可以称该子类叫finalize类，简称F类。

JVM会在子类实例化的时候创建一个Finalizer 对象与之关联。,finalize方法会在垃圾回收之前调用。


下面来看 Reference 的结构

    其内部提供2个构造函数，一个带queue，一个不带queue。其中queue的意义在于，我们可以在外部对这个queue进行监控。即如果有对象即将被回收，那么相应的reference对象就会被放到这个queue里。我们拿到reference，做一些其他操作。而如果不带的话，就只有不断地轮询reference对象，通过判断里面的get是否返回null(phantomReference对象不能这样作，其get始终返回null，因此它只有带queue的构造函数。这两种方法均有相应的使用场景，取决于实际的应用。如weakHashMap中就选择去查询queue的数据，来判定是否有对象将被回收。而ThreadLocalMap，则采用判断get()是否为null来作处理。


    public abstract class Reference<T> {

    // 引用的对象


    private T referent;         /* Treated specially by GC */

    ReferenceQueue<? super T> queue;

    Reference next;
    // discovered 字段
    transient private Reference<T> discovered;  /* used by VM */

    // 静态同步锁
    static private class Lock { };
    private static Lock lock = new Lock();
    //静态字段，虚拟机将要入队的引用对象放在这里，是一个链表
    private static Reference pending = null;

    private static class ReferenceHandler extends Thread {

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }
        // 入队操作。
        public void run() {
            for (;;) {

                Reference r;
                synchronized (lock) {
                    if (pending != null) {
                        r = pending;
                        Reference rn = r.next;
                        pending = (rn == r) ? null : rn;
                        r.next = r;
                    } else {
                        try {
                            lock.wait();
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }

                // Fast path for cleaners
                // 如果是Cleaner对象，需要调用其cleaner方法，Cleaner继承虚引用
                if (r instanceof Cleaner) {
                    ((Cleaner)r).clean();
                    continue;
                }

                ReferenceQueue q = r.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }

    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
    
    
     Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
    
  Reference 构造函数的时候可以指定一个  ReferenceQueue，这样当被引用的对象被回收的时候，引用对象会被加入到此队列中，方便后续的收尾工作。 
    
可以看到有一个优先级很高的静态线程，负责把引用对象入队。




一个Reference 有如下四个阶段。



- Active 新创建的此对象都是在此状态，当被引用的对象变为不可达的时候，垃圾回收器会改变引用对象的状态为Pending或者Inactive。

- Pending 一个引用对象正在等待进入  ReferenceQueue，如果引用对象没有ReferenceQueue不会经过此状态。 此时next==this

- Enqueued 一个引用对象创建的时候关联了一个ReferenceQueue，一旦引用对象进入队列，就变成此状态，当引用对象从队列中移除的时候，状态变为Inactive。如果引用对象没有ReferenceQueue不会经过此状态。 

- Inactive 一旦一个引用变成此状态就不会再变化。 此时next==this



#### 引用对象入队时机 


关于引用对象进入ReferenceQueue的时机，如下分析。


Fobject 是一个覆盖了finalize 方法的类。可以查看如下几种情况


        public class Fobject{

    ReferenceQueue queue;
    public Fobject(ReferenceQueue queue) {
        this.queue=queue;
    }
    @Override
    protected void finalize() throws Throwable {
        System.out.println("finalize before "+queue.poll());
        super.finalize();
        System.out.println("finalize");
        System.out.println("finalize after "+queue.poll());
    }
 }


下面是例子



        // 引用队列    
         ReferenceQueue<Object> rq=new ReferenceQueue(); 
       //PhantomReference<Fobject> pr = new PhantomReference<>(new Fobject(rq), rq);
       //SoftReference<Fobject> pr = new SoftReference<>(new Fobject(rq), rq);
       //WeakReference<Fobject> pr = new WeakReference<>(new Fobject(rq), rq);
       //PhantomReference<Object> pr = new PhantomReference<>(new Object(), rq);
       //SoftReference<Object> pr = new SoftReference<>(new Object(), rq);
       // WeakReference<Object> pr = new WeakReference<>(new Object(), rq);
        System.out.println(pr+", "+pr.get()+", "+rq.poll());
        System.gc();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(pr+", "+pr.get()+", "+rq.poll());
        

##### 虚引用         
        
    // 引用对象没有入队，被引用对象没有被gc， 被引用对象 重生了。因为进入finalizer 列表,等待执行Finalize方法，此时被Finalizer 静态对象强引用。  
    PhantomReference<Fobject> pr = new PhantomReference<>(new Fobject(), rq);   
        
    java.lang.ref.PhantomReference@28f553e3, null, null
    finalize before null
    finalize
    finalize after null
    java.lang.ref.PhantomReference@28f553e3, null, null
        
    
     // 引用对象进入队列了，被引用对象被gc了。当被引用对象被gc后，引用对象才进入队列。
    PhantomReference<Object> pr = new PhantomReference<>(new Object(), rq);
    java.lang.ref.PhantomReference@75dfb148, null, null
    java.lang.ref.PhantomReference@75dfb148, null, java.lang.ref.PhantomReference@75dfb148
    
#### 弱引用        
     
     // 弱引用,在调用finalize方法前已经入队。调用finalize对象后 被引用对象才进入队列。
     WeakReference<Fobject> pr = new WeakReference<>(new Fobject(), rq);
    
    java.lang.ref.WeakReference@28f553e3, com.zhaopeng.eagle.study.Fobject@2567117, null
    finalize before java.lang.ref.WeakReference@28f553e3
    finalize
    finalize after null
    java.lang.ref.WeakReference@28f553e3, null, null


    // 弱引用，在调用finalize方法前已经入队。调用finalize对象后 被引用对象才进入队列。
    WeakReference<Object> pr = new WeakReference<>(new Object(), rq);
    java.lang.ref.WeakReference@30f02a6d, java.lang.Object@67717334, null
    java.lang.ref.WeakReference@30f02a6d, null, java.lang.ref.WeakReference@30f02a6d





#### 关于引用对象入队总结：


SoftReference，WeakReference和强引用 只有在finalize 之前已经入RefenceQueue，

PhantomReference 在被引用的对象被GC时候进入RefenceQueue，




####  ReferenceQueue


被引用的对象在被GC后，引用对象需要进入一个队列(如果存在的情况下)。


ReferenceQueue数据结构如下


    public class ReferenceQueue<T> {

    /**
     * Constructs a new reference-object queue.
     */
    public ReferenceQueue() { }

    // 静态内部类，不入队。
    private static class Null extends ReferenceQueue {
        boolean enqueue(Reference r) {
            return false;
        }
    }

    static ReferenceQueue NULL = new Null();
    static ReferenceQueue ENQUEUED = new Null();

    static private class Lock { };
    private Lock lock = new Lock();
    private volatile Reference<? extends T> head = null;
    private long queueLength = 0;

    // 入队 Reference 的静态线程类会调用此方法。
    boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
        synchronized (r) {
            if (r.queue == ENQUEUED) return false;
            synchronized (lock) {
                // 设置已经入队的操作，
                r.queue = ENQUEUED;
                // 设置next字段，头插法。
                r.next = (head == null) ? r : head;
                head = r;
                queueLength++;
                if (r instanceof FinalReference) {
                    sun.misc.VM.addFinalRefCount(1);
                }
                lock.notifyAll();
                return true;
            }
        }
    }

    // 准备出队
    private Reference<? extends T> reallyPoll() {       /* Must hold lock */
        if (head != null) {
            Reference<? extends T> r = head;
            head = (r.next == r) ? null : r.next;
             // queu设置为null
            r.queue = NULL;
            // next 字段指向自己. 
            r.next = r;
            queueLength--;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(-1);
            }
            return r;
        }
        return null;
    }

  
     //出队，必须加锁，
    public Reference<? extends T> poll() {
        if (head == null)
            return null;
        synchronized (lock) {
            return reallyPoll();
        }
    }

  
    public Reference<? extends T> remove(long timeout)
        throws IllegalArgumentException, InterruptedException
    {
        if (timeout < 0) {
            throw new IllegalArgumentException("Negative timeout value");
        }
        synchronized (lock) {
            Reference<? extends T> r = reallyPoll();
            if (r != null) return r;
            for (;;) {
                lock.wait(timeout);
                r = reallyPoll();
                if (r != null) return r;
                if (timeout != 0) return null;
            }
        }
    }

   
    public Reference<? extends T> remove() throws InterruptedException {
        return remove(0);
    }

 }


#### 总结

本文介绍了Reference和ReferenceQueue ，以及引用对象进入ReferenceQueue 的时机。

































   
    
    
参考资料：
https://www.zhihu.com/question/49760047