#### 简介
1. ConcurrentHashMap  和Hashmap的区分主要在于它是线程安全的，采用锁分段技术。它使用了多个锁来控制对hash表的不同部分进行的修改。ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个Segment 就是一个hash table[] 数组，都会有自己的锁，这样可以实现多个线程在不同的segment上修改操作。


2. ConcurrentHashMap 的一些方法比如 size()和containsValue()，它们可能需要锁定整个表而而不仅仅是某个段，这需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。这里“按顺序”是很重要的，否则极有可能出现死锁。



#### 存储结构
ConcurrentHashMap 的分断锁，表示它采用了锁分离技术，在ConcurrentHashMap内部，包含多个
Segment数组，Segment实现了ReentrantLock ,所以说Segment本身就是一个锁，这样一个ConcurrentHashMap就包含了多个锁。
在Segment内部有一个 HashEntry<K,V>[]数组，是真正来存储数据的地方。

结构示意图如下：

![image](http://7x00ae.com1.z0.glb.clouddn.com/ConcurrentHashMap%20%E7%BB%93%E6%9E%84%E5%9B%BE.png)



#### 方法
下面ConcurrentHashMap常用的方法。


##### 初始化



        /**
        * initialCapacity 初始化容量
        * loadFactor 是扩容因子
        * concurrencyLevel 是初始化segment的个数的 级别
        */
      public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        
        //ssize 是真正的Segment的个数，这样能保证Segment的个数为2的倍数
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        //是参与计算segment 位置的个数
        this.segmentShift = 32 - sshift;
        // segmentMask 是用来计算Segment的位置。与操作
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //  c 表示每个Segment中table[]的最大长度级别  
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        // cap是真正的table[]长度
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        //默认状态下，只会创建segments数组的第0个元素，并且使用Unsafe装入数组
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }


##### put 方法

    /**
     *  ConcurrentHashMap 的put方法
     *
     */
     public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        //根据hash值计算到Segment的索引
        //调用ensureSegment 方法获取到对应的Segment，如果对于的Segment为null则会创建.
        //h >>> segmentShift 这里只保留参数计算的位数
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        //调用Segment的put方法    
        return s.put(key, hash, value, false);
    }
    /**
     *  Segment 的put方法
     *
     */
    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);
            V oldValue;
            // 先尝试给这个Segment上锁
            // 
            try {
                HashEntry<K,V>[] tab = table;
                //找到对应的所以
                int index = (tab.length - 1) & hash;
                HashEntry<K,V> first = entryAt(tab, index);
                //类似于hashmap中的获取key对应的value,遍历当前索引中的每一个entry
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                //解锁
                unlock();
            }
            return oldValue;
        }
    
        



##### get 方法

    /**
     *  Segment 的get方法
     *  先获取Segment，然后获取table对应的索引，最后获取entry，注意此时不需要加锁。
     */
    public V get(Object key) {
        Segment<K,V> s; 
        HashEntry<K,V>[] tab;
        int h = hash(key);
        // h >>> segmentShift 这里只保留参数计算的位数
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
    
#### size 方法

返回当前concurrentHashmap中数据的个数,size方法需要锁住全部的Segment，因此可能出现锁表失败的情况，这时候会尝试重试，并且会多次计算元素的个数。
这里至少计算2次，如果前面两次计算结果一样，则计算正确，不需要锁表，直接返回结果，
如果前面两次计算结果不相同，则需要锁表。

    public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        long sum;         // sum of modCounts
        long last = 0L;   // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (;;) {
                // 锁表
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                // 两次计算结果相同则跳出循环
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
        //解锁
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }
