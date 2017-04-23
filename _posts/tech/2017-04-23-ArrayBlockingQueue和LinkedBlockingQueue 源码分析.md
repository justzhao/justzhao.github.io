---
layout: post
title: 阻塞队列源码分析
category: 技术基础
tags:  ArrayBlockingQueue，LinkedBlockingQueue 
description: 阻塞队列
---

#### 阻塞队列简介

Java 中常用的阻塞队列有ArrayBlockingQueue和LinkedBlockingQueu，他们都 都实现了BlockingQueue接口。而BlockingQueue接口实现了BlockingQueue 接口。
BlockingQueue 接口代码。

    public interface BlockingQueue<E> extends Queue<E> {
    
     
     // 阻塞队列添加一个元素，成功返回true，失败返回false，当没有空间的时候抛出IllegalStateException，如果被添加的元素是null，就会抛出NullPointerException
    boolean add(E e);

   
     // 阻塞队列添加一个元素,如果被添加的元素是null，就会抛出NullPointerException
     //如果容量不够返回false
    boolean offer(E e);

    
      // 阻塞队列添加一个元素,如果队列容量不够一直等待，如果被添加的元素是null，就会抛出NullPointerException，如果等待过程中被中断抛出InterruptedException
    void put(E e) throws InterruptedException;

    
     
      // 阻塞队列添加一个元素，如果容量不够，则等待timeout
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;

   
     // 获取和删除队列头部，如果队列是空的就等待
    E take() throws InterruptedException;

    
     // 获取和删除队头元素，如果队列为空，则等待timeout
    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;

    
     // 返回当前阻塞队列的容量
    int remainingCapacity();

    
     //删除一个指定的对象
    boolean remove(Object o);
    }





#### ArrayBlockingQueue

ArrayBlockingQueue 是有界限的阻塞队列，底层是通过数组实现存储元素，是一个FIFO队列。使用

ReentrantLock 来保证线程安全。
关键代码如下：


    public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    

    // 存放元素的数组
    final Object[] items;

    // 队首索引
    int takeIndex;

    // 队尾索引
    int putIndex;

    // 元素个数
    int count;

    

    //同步锁和信号量
    final ReentrantLock lock;
  
    private final Condition notEmpty;
  
    private final Condition notFull;

    // Internal helper methods

   // 环形 加减
    final int inc(int i) {
        return (++i == items.length) ? 0 : i;
    }

 
    final int dec(int i) {
        return ((i == 0) ? items.length : i) - 1;
    }

   
    
    private static void checkNotNull(Object v) {
        if (v == null)
            throw new NullPointerException();
    }

    /**
    * 本类特有的方法
    */
    
     //插入一个元素  必须持有锁
    private void insert(E x) {
        items[putIndex] = x;
        putIndex = inc(putIndex);
        ++count;
        notEmpty.signal();
    }

    /**
     * Extracts element at current take position, advances, and signals.
     * Call only when holding lock.
     */
     // 弹出一个元素 必须持有锁
    private E extract() {
        final Object[] items = this.items;
        E x = this.<E>cast(items[takeIndex]);
        items[takeIndex] = null;
        takeIndex = inc(takeIndex);
        --count;
        notFull.signal();
        return x;
    }

    // 删除指定索引的元素
    void removeAt(int i) {
        final Object[] items = this.items;
        // 如果索引等于队尾索引直接删除
        if (i == takeIndex) {
            items[takeIndex] = null;
            takeIndex = inc(takeIndex);
        } else {
            // 否则 所有的元素向前移动一个位置
            for (;;) {
                int nexti = inc(i);
                if (nexti != putIndex) {
                    items[i] = items[nexti];
                    i = nexti;
                } else {
                    items[i] = null;
                    putIndex = i;
                    break;
                }
            }
        }
        --count;
        notFull.signal();
    }

    
    /**
    * 实现的抽象方法
    */
    
    
    public boolean add(E e) {
        return super.add(e);
    }

    
     // 队尾插入元素，如果满了直接返回false。
    public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                insert(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }

     // 队尾插入元素，如果满了等待，阻塞当前线程
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
    }

    //   // 队尾插入元素，如果满了先等待 timeout ,然后直接返回false。
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            insert(e);
            return true;
        } finally {
            lock.unlock();
        }
    }

    // 弹出队首，如果队列为空弹出null
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : extract();
        } finally {
            lock.unlock();
        }
    }
    
     // 弹出队首,如果队列为空就等待
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return extract();
        } finally {
            lock.unlock();
        }
    }

    // 弹出队首
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            return extract();
        } finally {
            lock.unlock();
        }
    }
    // 获取元素
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : itemAt(takeIndex);
        } finally {
            lock.unlock();
        }
    }


#### LinkedBlockingQueue

LinkedBlockingQueue  是以单链表为基础的阻塞队列，
内部结构有拿锁和放锁进行操作。

拿锁实例化出一个notEmpty的Condition,放锁实例化一个notFull的Condition,用于在队列容量满或空状态变化时候通知阻塞的线程。

下面来看关键源码


    public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    private static final long serialVersionUID = -6903933977591709194L;

    

    // 内部静态类，节点类
    static class Node<E> {
        E item;

        //后继
        Node<E> next;

        Node(E x) { item = x; }
    }

    // 队列容量，原子操作，
    private final AtomicInteger count = new AtomicInteger(0);

    // 队头节点
    private transient Node<E> head;

    //队尾节点
    private transient Node<E> last;

   
    // 拿锁和放锁，以及相关信号量
    private final ReentrantLock takeLock = new ReentrantLock();
   
   
    private final Condition notEmpty = takeLock.newCondition();

   
    private final ReentrantLock putLock = new ReentrantLock();

    
    private final Condition notFull = putLock.newCondition();

    // 私有方法,通知非满和非空
    
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

  
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }

    // 入队操作
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }

    // 出队操作
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }

    //全都锁住
    void fullyLock() {
        putLock.lock();
        takeLock.lock();
    }

    
    void fullyUnlock() {
        takeLock.unlock();
        putLock.unlock();
    }


    // 构造方法
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

  
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }

    // 传入一个集合，构造一个队列
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }


    // 返回当前队列元素个数
    public int size() {
        return count.get();
    }

   
     // 返回队列还有多少容量，一般是不可靠的，没有同步机制
    public int remainingCapacity() {
        return capacity - count.get();
    }

    // 插入一个元素，容量不够会等待
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }

    // 插入一个元素
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }

    
     // 插入一个元素，容量不够马上返回false
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }

    // 获取一个元素，队列空时候会等待
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

/ 获取队首元素，队列空的时候返回null，会删除队首
    public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

    // 获取队首元素，队列空的时候返回null。
    public E peek() {
        if (count.get() == 0)
            return null;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            Node<E> first = head.next;
            if (first == null)
                return null;
            else
                return first.item;
        } finally {
            takeLock.unlock();
        }
    }

    /**
     * Unlinks interior Node p with predecessor trail.
     */
    void unlink(Node<E> p, Node<E> trail) {
        // assert isFullyLocked();
        // p.next is not changed, to allow iterators that are
        // traversing p to maintain their weak-consistency guarantee.
        p.item = null;
        trail.next = p.next;
        if (last == p)
            last = trail;
        if (count.getAndDecrement() == capacity)
            notFull.signal();
    }

    
     // 删除一个指定的元素，有可能当前的队列会包含多个同样的元素，
     // 此时需要锁定拿锁和放锁
    public boolean remove(Object o) {
        if (o == null) return false;
        fullyLock();
        try {
            for (Node<E> trail = head, p = trail.next;
                 p != null;
                 trail = p, p = p.next) {
                if (o.equals(p.item)) {
                    unlink(p, trail);
                    return true;
                }
            }
            return false;
        } finally {
            fullyUnlock();
        }
    }

    
    // 是否包含某个元素，需要同时锁住拿锁和放锁。
    public boolean contains(Object o) {
        if (o == null) return false;
        fullyLock();
        try {
            for (Node<E> p = head.next; p != null; p = p.next)
                if (o.equals(p.item))
                    return true;
            return false;
        } finally {
            fullyUnlock();
        }
    }
    
#### 总结
    

LinkedBlockingQueue 内部需要两个锁的原因是，线程在入队或者出队的时候只需要分表操作tail节点和head节点，这时候使用两个锁分别同步能
保证入队操作和出对操作隔离。

ArrayBlockingQueue 初始化必须要知道数组的大小。