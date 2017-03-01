---
layout: post
title: Java 线程池
category: 技术基础
tags: Java ThreadPool  
description:  线程池

---

#### Java线程池

Java中常使用线程池来存放一些空闲线程，当有任务需要执行，就从一个线程池取一个线程来执行任务。

一个线程池应该有线程管理器，工作线程，任务队列。
需要提供管理线程的生命周期，任务队列主要是用来存储需要执行的任务。

下面是创建一个线程池的例子。

		ExecutorService	exec = new ThreadPoolExecutor(20,100, 1, TimeUnit.SECONDS,new 
	ArrayBlockingQueue<Runnable>(1000),Executors.defaultThreadFactory()，new ThreadPoolExecutor.AbortPolicy());
	                
	//corePoolSize 线程池中线程的个数，包括空闲线程
	//maximumPoolSize 线程池中线程的最大个数
	//keepAliveTime 当线程的个数超过corePoolSize的时候，线程等待获取任务的时间，超过这个时间
	//此线程就会终止，
	//unit 时间单位
	// workQueue 工作队列
	//threadFactory 线程工厂
	// 当队列和线程池都满了情况下对新任务的处理方式s
	public ThreadPoolExecutor(int corePoolSize,
	                              int maximumPoolSize,
	                              long keepAliveTime,
	                              TimeUnit unit,
	                              BlockingQueue<Runnable> workQueue,
	                              ThreadFactory threadFactory,
	                              RejectedExecutionHandler handler) {
	        if (corePoolSize < 0 ||
	            maximumPoolSize <= 0 ||
	            maximumPoolSize < corePoolSize ||
	            keepAliveTime < 0)
	            throw new IllegalArgumentException();
	        if (workQueue == null || threadFactory == null || handler == null)
	            throw new NullPointerException();
	        this.corePoolSize = corePoolSize;
	        this.maximumPoolSize = maximumPoolSize;
	        this.workQueue = workQueue;
	        this.keepAliveTime = unit.toNanos(keepAliveTime);
	        this.threadFactory = threadFactory;
	        this.handler = handler;
	    } 


当一个任务被提交到线程池的时候


- 如果此时线程池中的数量小于corePoolSize，即使线程池中的线程都处于空闲状态，也要创建新的线程来处理被添加的任务。
 
- 如果此时线程池中的数量等于 corePoolSize，但是缓冲队列 workQueue未满，那么任务被放入缓冲队列。
 
- 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量小于maximumPoolSize，建新的线程来处理被添加的任务。
 
- 如果此时线程池中的数量大于corePoolSize，缓冲队列workQueue满，并且线程池中的数量等于maximumPoolSize，那么通过 handler所指定的策略来处理此任务。也就是：处理任务的优先级为：核心线程corePoolSize、任务队列workQueue、最大线程 maximumPoolSize，如果三者都满了，使用handler处理被拒绝的任务。

- 线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止。这样，线程池可以动态的调整池中的线程数。


##### 线程池相关的类

在Java 中线程池相关的类如下所示

![image](http://7x00ae.com1.z0.glb.clouddn.com/%E7%BA%BF%E7%A8%8B%E6%B1%A0.png)

	 //只有一个执行任务的接口
	public interface Executor；

	// 扩充来Executor接口，新增submit，newTaskFor方法，可以返回Future结果
	public abstract class AbstractExecutorService implements ExecutorService;

	// 具体线程池的实现，新增生命周期管理，和线程池状态的变化，以及销毁方法
	public class ThreadPoolExecutor extends AbstractExecutorService {

	



##### 线程池的状态

- running 正常运行
- shutdown 不接受新的任务
- stop 不接受新的任务，不执行队列中的任务，中断正常执行的任务
- tidying 所有的任务结束，工作线程的个数为0
- terminated  线程处于销毁状态，terminated方法执行结束


常使用ThreadPoolExecutor 来实现自己的线程池，下面来分析它的源码

#### ThreadPoolExecutor 源码分析
ThreadPoolExecutor 继承AbstractExecutorService 而 AbstractExecutorService继承
ExecutorService接口，ExecutorService 接口扩展了Executor。
下面是对应的源码


	public class ThreadPoolExecutor extends AbstractExecutorService {

    //此变量是原子的，用于记录所有有效线程的数量和每个线程的状态。
    //低29存线程数，高3位存线程状态runState。
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //CAPACITY =00011111111111111111111111111111
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    
    
    //runState 的五个状态。
    //线程池正常运行，可以接受新的任务并处理队列中的任务；
    private static final int RUNNING    = -1 << COUNT_BITS;
    //不再接受新的任务，但是会执行队列中的任务；
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //不再接受新的任务，不执行队列中的任务，中断正在执行的任务
    private static final int STOP       =  1 << COUNT_BITS;
    //所有的任务结束，工作线程个数为0。
    private static final int TIDYING    =  2 << COUNT_BITS;
    // terminated()方法执行结束
    private static final int TERMINATED =  3 << COUNT_BITS;
    
    //取出runState的值 
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    //取出workerCount的值
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    //将runState和workerCount存到同一个int中
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    
    // runsate的比较方法
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }

    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }


    // 工作线程+1
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }
    
    // 工作现场-1
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }
    
     //工作线程-1 直到成功
     private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }


    //工作队列
    private final BlockingQueue<Runnable> workQueue;
    
    // 当线程池无法处理任务时候，默认丢弃任务。 
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
        
    //核心线程池大小，活动线程小于corePoolSize则直接创建，
    //大于等于则先加到workQueue中，队列满了才创建新的线程。
    private volatile int corePoolSize;
    // 线程最大数量
    private volatile int maximumPoolSize;
    
    private volatile boolean allowCoreThreadTimeOut;
    //获取队列中任务的超时时间
    private volatile long keepAliveTime;
    
    
    private int largestPoolSize;
     
    private final HashSet<Worker> workers = new HashSet<Worker>();
      
    // 用于线程池一些资源的竞争操作
    private final ReentrantLock mainLock = new ReentrantLock();
    
    
    private final Condition termination = mainLock.newCondition();
    
    
    //work 类，用于封装任务，本事也实现了AQS，表示他本身也是可充入锁。
    //也实现了runnable 接口
    //实现互斥锁主要目的是为了中断的时候判断线程是在空闲还是运行，
    //Worker之所以继承AbstractQueuedSynchronizer类的语义
    //也是为了保护一个正在等待执行任务的Worker线程不被中断操作影响
    private final class Worker extends AbstractQueuedSynchronizer implements Runnable
    {
    
        private static final long serialVersionUID = 6138294804551838833L;

        
        //执行任务的线程。
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        // 初始化work时候的任务。
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

      
        Worker(Runnable firstTask) {
			// 调用runWork之前，禁止中断
			// state 是aqs中共享资源，默认0
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        // worker的线程执行runWoker方法
        public void run() {
            runWorker(this);
        }

       
         
         //锁状态 0 代表 解锁，1代表锁住
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        // 尝试获取锁，
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        // 释放锁
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

    //下面是ThreadPoolExecutor中 任务执行的方法。
    
    
    
    //worker的run方法会调用此方法
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        // 在Woker的构造函数中设置了state=-1，抑制了线程中断，所以这里unlock恢复状态、
		// sate +1
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 如果当前task=null 则调用getTask获取任务,
            // 当getTask()为null的情况时候，会不断调用getTask直到线程减少
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                //如果线程池的状态大于stop，确保线程的状态为interrupted。
                
                if ((runStateAtLeast(ctl.get(), STOP) ||
    (Thread.interrupted() &&runStateAtLeast(ctl.get(), STOP))) &&!wt.isInterrupted())
                    wt.interrupt();
                try {
                    //任务执行前一些预处理方法
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 任务执行之后的一些方法
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            // 正常结束runWork
            completedAbruptly = false;
        } finally {
            // 处理完任务之后的一些清扫工作。
            processWorkerExit(w, completedAbruptly);
        }
    }
    
    // 综合需要终止线程和返回null的情况
    //1、当前活动线程数超过maximumPoolSize个（调用了setMaximumPoolSize的缘故）；
    //2、线程池已经停止（STOP）；
    //3、线程池已经关闭（SHUTDOWN）且任务队列为空；
    // 4、工作线程获取任务超时，且满足(allowCoreThreadTimeOut || workerCount >corePoolSize)条件
     private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 必要的状态检测   
            // 当线程池状态为shutdown时候 要么rs>=stop。
            //要么队列为空。此时减少工作线程的个数,并且直接返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
              
                decrementWorkerCount();
                return null;
            }

            boolean timed;      // Are workers subject to culling?

            for (;;) {
                int wc = workerCountOf(c);
                
                // 1.core thread允许被超时，那么超过corePoolSize的的线程必定有超时
                // 2.allowCoreThreadTimeOut == false && wc >
                // corePoolSize时，一般都是这种情况，core 
                // thread即使空闲也不会被回收，只要超过的线程才会
                timed = allowCoreThreadTimeOut || wc > corePoolSize;
            
                // 如果当前工作线程小于最大数，且timeout=false 或者timed=false                          // 就跳出循环，获取任务
                if (wc <= maximumPoolSize && ! (timedOut && timed))
                    break;
                // 否则减少工作线程的个数，并且返回。    
                if (compareAndDecrementWorkerCount(c))
                    return null;
                c = ctl.get();  // Re-read ctl
                
                // 重新获取线程状态，如果被改变就需要重新检测
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }

            try {
                // 1.以指定的超时时间从队列中取任务
                // 2.core thread没有超时
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                //线程中断
                timedOut = false;
            }
        }
    }
    // 线程和数据情况
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
     // 异常结束runWork时候completedAbruptly=true直接减少线程个数。
     // 正常结束runWork时候completedAbruptly=false。
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            //移除工作线程
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        //尝试结束，
        tryTerminate();


        //当池的状态还是RUNNING，
        //又要分两种情况，一种是异常结束，一种是正常结束。异常结束比较好弄，直接加个线程
        //替换死掉的线程就好了，也就是最后的addWorker操作。
        //而正常结束又有几种情况了，如果允许core线程超时，也就是allowCoreThreadTimeOut为
        //true，那么在池中没有任务的时候，调用带有时限参数的poll方法时就可能返回null，
        //致使线程正常退出，如果允许core线程超时，池中最小的线程数可为0，如果此时队列又/         //有任务了，那么池中必须要有一个线程，若池中活动的线程数不为0，
        //就不需要新增线程来替换死掉的线程，否则就要新增一个；
        //如果不允许core线程超时，池中的线程必须达到corePoolSize个才能让多的线程退出，
        //而不需要用新的线程替换，否则也需要新增一个线程替换这个死掉的线程。
            
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            
            // 如果runWork 正常结束
            if (!completedAbruptly) {
                // 检测是否超时
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
    
    //所有中断线程
     private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
    
     private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

    
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
   
    
    
    
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
      
         // 分为3个步骤
         // 1 如果当前运行线程数少于 corePoolSize.尝试新建一个线程和task封装成一个Worker
         // 会调用addWord方法，检测runState和workerCount的状态。
         
        // 2 当运行线程大于corePoolSize时候，这时候就把任务放入队列中，然后再次检查
        // 非RUNNING状态 则从workQueue中移除任务并拒绝,如果是运行状态或者移除任务失败，
        // 加入一个空的work。
        // 3 如果我们无法把任务加入队列 ，就新建线程，如果失败，就拒绝执行任务。（非RUNNING状态拒绝新的任务，队列满了启动新的线程失败）
        
        int c = ctl.get(); 
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 如果是运行状态而且 任务加入队列成功
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
            //防止SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况
                addWorker(null, false);
        }// 进入队列失败，然后新建线程
        else if (!addWorker(command, false))
            reject(command);
    }
    
    // 添加一个worker 任务
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 检测状态，如果 当前状态大于0，非running状态。
            //   且不是shutDown状态，且任务不为空，且工作队列是空。
            // 就直接返回false，没必要加work。
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN && firstTask == null &&! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                 // 如果当前work数量大于极限
                 // 或者在core=true情况下。大于 核心线程个数。
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 增加worker数量成功跳出for循环    
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                 //如果线程池的状态发生变化则重试 ,重复上述工作
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock;
            // 新建一个worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();
                    int rs = runStateOf(c);
                    /// RUNNING状态 || SHUTDONW状态下清理队列中剩余的任务
                    if (rs < SHUTDOWN ||(rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                            
                        //如果满足条件就加入线程池中   
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 如果加入成功就开启线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
             //加入失败，调用addWorkFailed方法
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
    
    // 添加worker失败后的清理工作，比如计数器-1，移除任务等。
     private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }

    // 接下来是线程池状态改变的一些方法。
    
    
    //先看 runState 的五个状态。
    //线程池正常运行，可以接受新的任务并处理队列中的任务；
    private static final int RUNNING    = -1 << COUNT_BITS;
    //不再接受新的任务，但是会执行队列中的任务；
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //不再接受新的任务，不执行队列中的任务，中断正在执行的任务
    private static final int STOP       =  1 << COUNT_BITS;
    //所有的任务结束，工作线程个数为0。
    private static final int TIDYING    =  2 << COUNT_BITS;
    // terminated()方法执行结束
    private static final int TERMINATED =  3 << COUNT_BITS;
    /**
     * Initiates an orderly shutdown in which previously submitted
     * tasks are executed, but no new tasks will be accepted.
     * Invocation has no additional effect if already shut down.
     *
     * <p>This method does not wait for previously submitted tasks to
     * complete execution.  Use {@link #awaitTermination awaitTermination}
     * to do that.
     *
     * @throws SecurityException {@inheritDoc}
     */
     // 尝试把线程池状态修改到SHUTDOWN，中断空闲线程，不接受新的任务，
     //执行已经在队列中的任务，此方法不会等待正在执行的任务结束，
     // 最后检测线程池个数是否为0，并且切换到TERMINATED状态。
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //SecurityManager 检测是否有权限
            checkShutdownAccess();
            // 改变状态
            advanceRunState(SHUTDOWN);
            // 中断空闲线程
            interruptIdleWorkers();
            //ScheduledThreadPoolExecutor 中的方法 根据关闭规则是否清理队列中的任务
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
    
    // 中断空闲线程
     private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }
    // onlyOne为ture是表示最多中断一个线程
    // 如果一个线程不是已经中断状态和非上锁状态，就中断它
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
    
    //尝试把状态改变为TIDYING
    
    
    //这里调用了一个interruptIdleWorkers(ONLY_ONE)操作去中断一个空闲线程。
    //这么做是为什么？【关于这个的理解可能有问题】调用这个方法的目的是将shutdown信号传播给其它线程。
    //调用shutdown方法的时候会去中断所有空闲线程，如果这时候池中所有的线程都正在执行任务，那么就不会有线程被中断，
    //调用shutdown方法只是设置了线程池的状态为SHUTDOWN，在取任务(getTask,后面会细说)的时候，
    //假如很多线程都发现队列里还有任务（没有使用锁，存在竞态条件），然后都去调用take，如果任务数小于池中的线程数，
    //那么必然有方法调用take后会一直等待（shutdown的时候这些线程正在执行任务，所以没能调用它的interrupt，其中断状态没有被设置），
    //那么在没有任务且线程池的状态为SHUTDWON的时候，这些等待中的空闲线程就需要被终止iinterruptIdleWorkers(ONLY_ONE)回去中断一个线程，
    //让其从take中退出，然后这个线程也进入同样的逻辑，去终止一个其它空闲线程，直到池中的活动线程数为0。
    
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 如果线程池状态为运行，或者状态大于 TIDYING（已经teminate） 
            // 或者线程状态是shutdown 但是工作队列不为空，就直接返回
            //满足终止条件的因素有两个：首先，ctl状态为STOP,或者为SHUTDOWN且任务队列为空
            if (isRunning(c) || runStateAtLeast(c, TIDYING) || (runStateOf(c) ==SHUTDOWN&& ! workQueue.isEmpty()))
                return;
            // 如果工作线程的个数不等于0，就先中断一个空闲线程，然后返回    
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                 //设置TIDYING状态成功
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                     // 调用钩子方法
                        terminated();
                    } finally {
                    //设置为TERMINATED状态
                        ctl.set(ctlOf(TERMINATED, 0));
                    //awaitTermination 方法会调用await
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }

    
     
     // 停止正在执行的任务和待执行的任务，并且返回队列中待执行的方法
     // 任务队列会被清空
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            // 清空并且返回队列
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        // 尝试结束线程池
        tryTerminate();
        return tasks;
    }
    
     // 清除任务队列，并且返回队列中的任务
      private List<Runnable> drainQueue() {
        BlockingQueue<Runnable> q = workQueue;
        List<Runnable> taskList = new ArrayList<Runnable>();
        q.drainTo(taskList);
        if (!q.isEmpty()) {
            for (Runnable r : q.toArray(new Runnable[0])) {
                if (q.remove(r))
                    taskList.add(r);
            }
        }
        return taskList;
    }
    
    /**
     * Blocks until all tasks have completed execution after a shutdown
     * request, or the timeout occurs, or the current thread is
     * interrupted, whichever happens first.
     */
    //会阻塞所有完成任务后的线程，直到线程池关闭，或者超时，或者中断。
    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }

    //是否为非running状态
    public boolean isShutdown() {
        return ! isRunning(ctl.get());
    }

    /**
     * Returns true if this executor is in the process of terminating
     * after {@link #shutdown} or {@link #shutdownNow} but has not
     * completely terminated.  This method may be useful for
     * debugging. A return of {@code true} reported a sufficient
     * period after shutdown may indicate that submitted tasks have
     * ignored or suppressed interruption, causing this executor not
     * to properly terminate.
     *
     * @return true if terminating but not yet terminated
     */
     //如果关闭后所有任务都已完成，则返回 true。
     //注意，除非首先调用 shutdown 或 shutdownNow，否则 isTerminated 永不为 true。
    
    public boolean isTerminating() {
        int c = ctl.get();
        return ! isRunning(c) && runStateLessThan(c, TERMINATED);
    }
    
	}



#### 总结

文章介绍了线程池的概念和一些基本的原理。
线程池有五个状态和在执行任务的过程中线程池状态的变化。
介绍了当执行一个新任务的时候，线程池发生了什么。
关于线程池还有ForkJoinPool ,ScheduledExecutorService，Executors，这些类也需要着重理解。


