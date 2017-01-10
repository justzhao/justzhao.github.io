---
layout: post
title: Java 引用类型(二 FinalReference和Finalizer)
category: 技术基础
tags: Java 引用类型(二 FinalReference和Finalizer)
description:  Java 引用类型(二 FinalReference和Finalizer)
---

#### 简介

在Java中FinalReference就是强引用类型，FinalReference 作为 java.lang.ref 里的一个不能被公开访问的类，源码如下：

    class FinalReference<T> extends Reference<T> {

    public FinalReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

    }
    
可以看到   FinalReference 非常简单，直接继承Reference类。与之相关的Finalizer是FinalReference的实现类，它的访问权限也是非公开的，源码如下：
    
    
    final class Finalizer extends FinalReference { /* Package-private; must be in
                                                  same package as the Reference
                                                  class */

    private static ReferenceQueue queue = new ReferenceQueue();
    private static Finalizer unfinalized = null;
    private static final Object lock = new Object();

    private Finalizer
        next = null,
        prev = null;

    private boolean hasBeenFinalized() {
        return (next == this);
    }

    private void add() {
        synchronized (lock) {
            if (unfinalized != null) {
                this.next = unfinalized;
                unfinalized.prev = this;
            }
            unfinalized = this;
        }
    }

    private void remove() {
        synchronized (lock) {
            if (unfinalized == this) {
                if (this.next != null) {
                    unfinalized = this.next;
                } else {
                    unfinalized = this.prev;
                }
            }
            if (this.next != null) {
                this.next.prev = this.prev;
            }
            if (this.prev != null) {
                this.prev.next = this.next;
            }
            this.next = this;   /* Indicates that this has been finalized */
            this.prev = this;
        }
    }

    private Finalizer(Object finalizee) {
        super(finalizee, queue);
        add();
    }

    /* Invoked by VM */
    static void register(Object finalizee) {
        new Finalizer(finalizee);
    }

    private void runFinalizer(JavaLangAccess jla) {
        synchronized (this) {
            if (hasBeenFinalized()) return;
            remove();
        }
        try {
            Object finalizee = this.get();
            if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
                jla.invokeFinalize(finalizee);

                /* Clear stack slot containing this variable, to decrease
                   the chances of false retention with a conservative GC */
                finalizee = null;
            }
        } catch (Throwable x) { }
        super.clear();
    }

    /* Create a privileged secondary finalizer thread in the system thread
       group for the given Runnable, and wait for it to complete.

       This method is used by both runFinalization and runFinalizersOnExit.
       The former method invokes all pending finalizers, while the latter
       invokes all uninvoked finalizers if on-exit finalization has been
       enabled.

       These two methods could have been implemented by offloading their work
       to the regular finalizer thread and waiting for that thread to finish.
       The advantage of creating a fresh thread, however, is that it insulates
       invokers of these methods from a stalled or deadlocked finalizer thread.
     */
    private static void forkSecondaryFinalizer(final Runnable proc) {
        AccessController.doPrivileged(
            new PrivilegedAction<Void>() {
                public Void run() {
                ThreadGroup tg = Thread.currentThread().getThreadGroup();
                for (ThreadGroup tgn = tg;
                     tgn != null;
                     tg = tgn, tgn = tg.getParent());
                Thread sft = new Thread(tg, proc, "Secondary finalizer");
                sft.start();
                try {
                    sft.join();
                } catch (InterruptedException x) {
                    /* Ignore */
                }
                return null;
                }});
    }

    /* Called by Runtime.runFinalization() */
    static void runFinalization() {
        if (!VM.isBooted()) {
            return;
        }

        forkSecondaryFinalizer(new Runnable() {
            private volatile boolean running;
            public void run() {
                if (running)
                    return;
                final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
                running = true;
                for (;;) {
                    Finalizer f = (Finalizer)queue.poll();
                    if (f == null) break;
                    f.runFinalizer(jla);
                }
            }
        });
    }

    /* Invoked by java.lang.Shutdown */
    static void runAllFinalizers() {
        if (!VM.isBooted()) {
            return;
        }

        forkSecondaryFinalizer(new Runnable() {
            private volatile boolean running;
            public void run() {
                if (running)
                    return;
                final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
                running = true;
                for (;;) {
                    Finalizer f;
                    synchronized (lock) {
                        f = unfinalized;
                        if (f == null) break;
                        unfinalized = f.next;
                    }
                    f.runFinalizer(jla);
                }}});
    }

    private static class FinalizerThread extends Thread {
        private volatile boolean running;
        FinalizerThread(ThreadGroup g) {
            super(g, "Finalizer");
        }
        public void run() {
            if (running)
                return;

            // Finalizer thread starts before System.initializeSystemClass
            // is called.  Wait until JavaLangAccess is available
            while (!VM.isBooted()) {
                // delay until VM completes initialization
                try {
                    VM.awaitBooted();
                } catch (InterruptedException x) {
                    // ignore and continue
                }
            }
            final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
            running = true;
            for (;;) {
                try {
                    Finalizer f = (Finalizer)queue.remove();
                    f.runFinalizer(jla);
                } catch (InterruptedException x) {
                    // ignore and continue
                }
            }
        }
    }

    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread finalizer = new FinalizerThread(tg);
        finalizer.setPriority(Thread.MAX_PRIORITY - 2);
        finalizer.setDaemon(true);
        finalizer.start();
    }

    }
    
####  FinalReference和Finalizer的关系

##### Finalizer 类

Finalizer 是FinalReference的一个实现。

Java 中 Object类是所有类的父类在Object类中有一个finalize方法

    protected void finalize() throws Throwable { }

如果子类覆盖了finalize的方法，可以称该子类叫finalize类，简称F类,finalize方法会在垃圾回收之前调用。

如果某个类实现了finalize方法，则会随之实例化一个Finalizer对象。

判断当前类是否是一个f类的标准要求是finalize方法必须非空.

Finalizer  主要在垃圾回收过程中会调用FinalizerThread做回收工作。



##### Finalizer 的实例化

可以看到Finalizer的构造方法是私有的。所以无法直接实例化。

如果某个类实现了finalize方法，虚拟机在对象实例化的时候会随之实例化一个Finalizer对象。

Finalizer 会在对应的FinalReference对象实例化的时候由JVM调用register方法(对象空间分配好之后就调用register方法)，从而调用私有的构造方法。
    
    private Finalizer(Object finalizee) {
        super(finalizee, queue);
        add();
    }    

super 方法调用父类的构造方法，并且传递当前的静态ReferenceQueue。

add()方法会把当前对象对象加入静态unfinalized 双向链表(Finalizer对象链)，此时当前对象会一直存在在Finalizer对象链中。


##### Finalizer 回收

在源码中可以看到如下静态线程，设置为守护线程并且启动。

     static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread finalizer = new FinalizerThread(tg);
        finalizer.setPriority(Thread.MAX_PRIORITY - 2);
        finalizer.setDaemon(true);
        finalizer.start();
    }


垃圾回收线程， Finalizer f = (Finalizer)queue.remove(); 可以看到不断的从静态queue中获取对象，然后调用它的runFinalizer方法。
在runFinalizer方法中会调用remove方法，把当前对象从Finalizer对象链移除，并且调用
 jla.invokeFinalize(finalizee),invokeFinalizeMethod方法就是调了当前对象的finalize方法。

    private static class FinalizerThread extends Thread {
        private volatile boolean running;
        FinalizerThread(ThreadGroup g) {
            super(g, "Finalizer");
        }
        public void run() {
            if (running)
                return;

            // Finalizer thread starts before System.initializeSystemClass
            // is called.  Wait until JavaLangAccess is available
            while (!VM.isBooted()) {
                // delay until VM completes initialization
                try {
                    VM.awaitBooted();
                } catch (InterruptedException x) {
                    // ignore and continue
                }
            }
            final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
            running = true;
            for (;;) {
                try {
                    Finalizer f = (Finalizer)queue.remove();
                    f.runFinalizer(jla);
                } catch (InterruptedException x) {
                    // ignore and continue
                }
            }
        }
    }
    
    
##### Finalizer 对象何时进入queue
在Java虚拟机中
当对象变成(GC Roots)不可达时，GC会判断该对象是否覆盖了finalize方法，若未覆盖，则直接将其回收。否则，若对象未执行过finalize方法，将其放入queue队列，


#### 总结

在Finalizer 的父类 Reference有一个静态线程会不断把不可达的对象加入queue。

而在Finalizer 中静态线程不断从queue中移除对象，执行对应的finalize方法。

而这两个线程的优先级很低，执行时机很不确定，所以一般不在finalize方法中做内存回收的操作。