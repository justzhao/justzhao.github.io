---
layout: post
title: 非阻塞队列分析
category: 技术基础
tags: 非阻塞, cas
description: 非阻塞队列
---
#### 简介
Java 提供了线程安全的阻塞队列和非阻塞队列，其中JDK中实现的非阻塞队列是ConcurrentLinkedQueue.

队列中元素按FIFO原则进行排序．使用链表的实现的无界队列，
采用wait-free算法来保证元素的一致性和线程安全.


#### 结构

![image](http://7x00ae.com1.z0.glb.clouddn.com/ConcurrentLinkedQueue%20%E7%BB%93%E6%9E%84.jpg)

如图所示ConcurrentLinkedQueue 内部声明了一个节点类Node，Node包含一些基本的操作。

默认的构造函数如下

    private transient volatile Node<E> head;
    
    private transient volatile Node<E> tail;

     public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }

#### Node 类解析

    private static class Node<E> {
        // item 可能为null
        volatile E item;
        volatile Node<E> next;

       
         // 构造方法，使用UNSAFE.putObject 
        Node(E item) {
            //设置obj对象中offset偏移地址对应的object型field的值为指定值
            UNSAFE.putObject(this, itemOffset, item);
        }

        boolean casItem(E cmp, E val) {
            return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        /**
        * putOrderedObject 意思：
        *按照order来进行写; 在底层操作时使用 store-store屏障,不会被重排序;
        *   putOrderedObject 使用 store-store barrier屏障, 而 putObject还会使用
        * store-load barrier 屏障，store-load barrier 屏障最消耗性能。
        *
        *   lazySet()的使用的道理，就是在不需要让共享变量的修改立刻让其他线程可见的时候，以设置普通变量的方式来修改共享状态，可以减少不必要的内存屏障，从而提高程序执行的效率。
        *
        */
        void lazySetNext(Node<E> val) {
            //设置后继节点
            // 可能会有点延时
            UNSAFE.putOrderedObject(this, nextOffset, val);
        }

        boolean casNext(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        // Unsafe mechanics

        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class k = Node.class;
                itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
    
    
#### 入队

ConcurrentLinkedQueue 的入队操作和常见的链表操作区别很大，在入队操作中，会可能有两次CAS操作。

如下图来自《java 并发编程的艺术》所示，表示了四个节点入队过程中，tail节点的变化。

![image](http://7x00ae.com1.z0.glb.clouddn.com/ConcurrentLinkedQueue%20%E5%85%A5%E9%98%9F%E6%93%8D%E4%BD%9C.jpg)

- 初始状态

      head = tail = new Node<E>(null);

也就是tail和head指向同一个空节点.      

- 添加元素1

    
在tail节点后面加上node1，tail 不变    
    
    
    
- 添加元素2

在node1 后面加上node2，tail指向node2


- 添加元素3

在node2后面加上node3，tail不变

- 添加元素4

在node3后面加上node4，tail 指向node4


整个入队过程中发现，tail并不会一直指向队列的最后一个节点。

下面结合入队代码来看。

入队操作的代码如下所示

    public boolean offer(E e) {
        //检测非null元素
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);
        // 操作0
        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            // 操作1
            if (q == null) {
                //操作2
                if (p.casNext(null, newNode)) {
                    //操作3
                    if (p != t) 
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
              
            }// 操作4
            else if (p == q)
                //操作5    
                p = (t != (t = tail)) ? t : head;
            else
                // 操作6
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }

###### 添加元素1

 1. 操作0,拿到当前tail节点，用变量p，t保存，且得到p的后继q
 2. 此时q一定是等于null的，然后执行操作3.cas添加节点，成功之后到操作3
 3. 执行操作3，此时p==t ，然后返回true。插入结束。


###### 添加元素2
 1. 同添加元素1的第一步。
 2. 此时q！=null直接到操作4
 3. 操作4 判断p是否等于q。这里需要结合出队操作来看。此时设定p!=q,到操作6
 4. 此时p==t，所以得到p=q,p节点后移动一位。开始新一次的循环。
 5. 此时q=p.next. q为null。依次执行添加元素1的第二步，第三步，插入结束，返回true。



上述描述是在单线程的情况，如果在多线程的情况就就不一样 了。

1. 初始阶段，当两个线程同时添加node1，和node2.

2. 假设线程1在操作2 成功cas，插入node1，线程2失败，跳到操作6. p=q之后开始新的循环。 此时来了一个线程3插入node3.并且抢先线程2先插入node3，注意此时会更新tail节点。
3. 线程2在操作6，此时p!=t，有如下代码
    
    t != (t = tail) // 此代码会判断tail节点是否变化过，t=tail 可以看成生成了一个临时变量。

此时tail引用已经被线程3改变， t != (t = tail) 成立，得到p=t。也就是p指向tail节点。开始新的一次循环。


可以看到，并不是每次入队操作都会有两次cas，这些就提高了队列的性能。




#### 出队

同样从《java并发编程的艺术》上的节点出队列图如下。


![image](http://7x00ae.com1.z0.glb.clouddn.com/ConcurrentLinkedQueue%20%E5%87%BA%E9%98%9F%E6%93%8D%E4%BD%9C.jpg)


- 初始状态

      包含head节点和四个元素节点
      

- 弹出元素1

元素1被弹出，head节点指向节点2

- 弹出元素2

节点2 的item被置为null，head节点不动

- 弹出元素3

节点3被弹出，head指向节点4

由上面出队顺序可以看见，head节点不一定一直指向一个item=null的节点，下面是出队的源码。

      public E poll() {
        // 操作0
        restartFromHead:
        for (;;) {
             // 操作1
            for (Node<E> h = head, p = h, q;;) {
                // 操作2
                E item = p.item;
                if (item != null && p.casItem(item, null)) {
                    // 操作3
                    if (p != h) // hop two nodes at a time
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                // 操作4
                else if ((q = p.next) == null) {
                    // 操作5
                    updateHead(h, p);
                    return null;
                }
                // 操作6
                else if (p == q)
                    continue restartFromHead;
                // 操作7
                else
                    p = q;
            }
        }
    }
    // 操作8
    final void updateHead(Node<E> h, Node<E> p) {
        if (h != p && casHead(h, p))
            h.lazySetNext(h);
    }

#### 弹出元素1

1. 到达操作0，记录一个跳转点
2. 操作2，记录head节点 h,p,判断当前p节点，也就是head的item是否为null，如果不为null，cas设置为null，
3. 如果cas成功，到达操作3,此时p=h。直接返回item，弹出元素1结束。
4. 
#### 弹出元素2

1. 同当初元素去的第一步
2. 操作2，记录head节点 h,p .此时p.item=null 为真。
3. 到达操作4，此时(q=p.next)==null 为假
4. 到达操作6 p==q 为假
5. 到达操作7， p=q, 此时开始第二轮循环。
6. 此时到达操作2,同第二步，此时p.item!=null 为真，并且cas设置item=null
7. 到达操作3此时p!=h为真，调用操作8 ，然后返回item

下面来解析 updateHead方法
    
    此时h！=p为真，且把p设置为头节点，然后调用 h.lazySetNext(h)，原来的头节点的后继设置为自己，即h.next=h 。这里使用lazySetNext 原因是这个操作并不需要立即对其他线程可见。
    

下面是多线程的情况：

假设有线程1,线程2 同时poll。在操作2。线程1成功cas，并且成功执行updateHead操作，
此时线程2还在操作2，直接到操作4， 计算q=p.next 。 注意此时p.next=p 因为被线程1改变了，
这时执行操作6，直接continue循环体外部，开始新的循环。

同样在offer操作中，因为出队操作也可能存在p.next=p的情况，这时候p点需要被回收，所以需要直接找到tail节点。

#### 总结

jdk 中的非阻塞队列使用cas 实现，根据Michael-Scott提出的非阻塞链接队列算法的基础上修改而来。是一个FIFO队列，
并不是每次的入队和出队的操作都会影响tail和head节点的移动，这样能提供性能，此外还使用了lazySet操作，减少内存屏障，提高性能。
    