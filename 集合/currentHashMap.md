#### 一、为什么使用CurrentHashMap
- 在多线程环境中使用HashMap的put方法有可能形成环形链表，即链表的一个节点的next节点永不为null，就会产生死循环；
- HashTable通过synchronized保证线程安全，效率太低
#### 二、jdk 8 改进
- ConcurrentHashMap 的结构从ReentrantLock+Segment+HashEntry变化为synchronized+CAS+HashEntry+红黑树
#### 三、重要成员变量
- 1、table:Node数组，通过volatile修饰，保证扩容时其它线程可见;
- 2、nextTable：哈希表扩容时生成的数据，数组为扩容前的2倍
- 3、sizeCtl:多个线程的共享变量，是操作的控制标识符，它的作用不仅包括threshold的作用，在不同的地方有不同的值也有不同的用途
     - 1、 －1 代表正在初始化
     - 2、 －N 代表N－1个线程正在进行扩容操作
     - 3、 0代表hash表还没有被初始化
     - 4、 正数表示下一次进行扩容的容量大小
- 4、ForwardingNode:Hash地址为-1，存储着nextTable的引用，只有table发生扩用的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或者已被移动;
- 5、Node
```java
static class Node<K,V> implements Map.Entry<K,V> {
       //key用final修饰
       final int hash;
       final K key;
       //用volatile修饰val
       volatile V val;
       volatile Node<K,V> next;

       Node(int hash, K key, V val, Node<K,V> next) {
           this.hash = hash;
           this.key = key;
           this.val = val;
           this.next = next;
       }

       public final K getKey()       { return key; }
       public final V getValue()     { return val; }
       public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
       public final String toString(){ return key + "=" + val; }
       public final V setValue(V value) {
           throw new UnsupportedOperationException();
       }

       public final boolean equals(Object o) {
           Object k, v, u; Map.Entry<?,?> e;
           return ((o instanceof Map.Entry) &&
                   (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                   (v = e.getValue()) != null &&
                   (k == key || k.equals(key)) &&
                   (v == (u = val) || v.equals(u)));
       }

       /**
        * Virtualized support for map.get(); overridden in subclasses.
        */
       Node<K,V> find(int h, Object k) {
           Node<K,V> e = this;
           if (k != null) {
               do {
                   K ek;
                   if (e.hash == h &&
                       ((ek = e.key) == k || (ek != null && k.equals(ek))))
                       return e;
               } while ((e = e.next) != null);
           }
           return null;
       }
   }
```
#### 四、构造函数
```java
/**
   * 无参构造函数，什么也不做，table的初始化放在了第一次插入数据时，默认容量大小是16和HashMap的一样，默认sizeCtl为0
   */
  public ConcurrentHashMap() {
  }

  /**
   * 传入容量大小的构造函数
   */
  public ConcurrentHashMap(int initialCapacity) {
      //如果传入的容量大小小于0 则抛出异常
      if (initialCapacity < 0)
          throw new IllegalArgumentException();
      /*
       * 如果传入的容量大小大于允许的最大容量值 则cap取允许的容量最大值 否则cap =
       * ((传入的容量大小 + 传入的容量大小无符号右移1位 + 1)的结果向上取最近的2幂次方)，
       * 即如果传入的容量大小是12 则 cap = 32(12 + (12 >>> 1) + 1=19
       * 向上取2的幂次方即32)，这里为啥一定要是2的幂次方，原因和HashMap的threshold一样，都是为
       * 了让位运算和取模运算的结果一样。
       * MAXIMUM_CAPACITY即允许的最大容量值 为2^30
       */
      int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                 MAXIMUM_CAPACITY :
                 //实现了将一个整数取2的幂次方
                 tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
      //将上面计算出的cap 赋值给sizeCtl，注意此时sizeCtl为正数，代表进行扩容的容量大小
      this.sizeCtl = cap;
  }

  /**
   * 包含指定Map的构造函数。
   * 置sizeCtl为默认容量大小 即16
   */
  public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
      this.sizeCtl = DEFAULT_CAPACITY;
      putAll(m);
  }

  /**
   * 传入容量大小和负载因子的构造函数。
   * 默认并发数大小是1
   */
  public ConcurrentHashMap(int initialCapacity, float loadFactor) {
      this(initialCapacity, loadFactor, 1);
  }

  /**
   * 传入容量大小、负载因子和并发数大小的构造函数
   */
  public ConcurrentHashMap(int initialCapacity,
                           float loadFactor, int concurrencyLevel) {
      if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
          throw new IllegalArgumentException();
      //如果传入的容量大小 小于 传入的并发数大小，
      //则容量大小取并发数大小，这样做的原因是确保每一个Node只会分配给一个线程，而一个线程则
      //可以分配到多个Node，比如当容量大小为64，并发数大
      //小为16时，则每个线程分配到4个Node
      if (initialCapacity < concurrencyLevel)   // Use at least as many bins
          initialCapacity = concurrencyLevel;   // as estimated threads
      long size = (long)(1.0 + (long)initialCapacity / loadFactor);
      int cap = (size >= (long)MAXIMUM_CAPACITY) ?
          MAXIMUM_CAPACITY : tableSizeFor((int)size);
      this.sizeCtl = cap;
  }
```
#### 五、重要方法
##### 5.1 put ＆ putval方法
- 1、判断键值是否为null，为null抛出异常；
- 2、调用spread（）计算key的hashCode;
- 3、如果当前table为空，则初始化table，通过cas保证只有一个线程执行初始化过程；
- 4、使用 容量大小-1 & 哈希地址 计算出待插入键值的下标，如果该下标上的bucket为null，则直接调用实现CAS原子性操作的casTabAt()方法将节点插入到table中，如果插入成功则完成put操作，结束返回。插入失败(被别的线程抢先插入了)则继续往下执行;
- 5、如果该下标上的节点(头节点)的哈希地址为-1，代表需要扩容，该线程执行helpTransfer()方法协助扩容；
- 6、如果该下标上的bucket不为空，且又不需要扩容，则进入到bucket中，同时锁住这个bucket（只是锁住该下标上的bucket而已，其他的bucket并未加锁，其他线程仍然可以操作其他未上锁的bucket）;
- 7、进入到bucket里面，首先判断这个bucket存储的是红黑树还是链表；
- 8、如果是链表，则遍历链表看看是否有哈希地址和键key相同的节点，有的话则根据传入的参数进行覆盖或者不覆盖，没有找到相同的节点的话则将新增的节点插入到链表尾部。如果是红黑树，则将节点插入。到这里结束加锁;
- 9、最后判断该bucket上的链表长度是否大于链表转红黑树的阈值(8)，大于则调用treeifyBin()方法将链表转成红黑树，以免链表过长影响效率；
- 10、调用addCount()方法，作用是将ConcurrentHashMap的键值对数量+1，还有另一个作用是检查ConcurrentHashMap是否需要扩容
```java
public V put(K key, V value) {
       return putVal(key, value, false);
   }

   /** Implementation for put and putIfAbsent */
   final V putVal(K key, V value, boolean onlyIfAbsent) {
       //判断键值是否为null，为null抛出异常
       if (key == null || value == null) throw new NullPointerException();
       //调用spread()方法计算key的hashCode()获得哈希地址
       int hash = spread(key.hashCode());
       int binCount = 0;
       for (Node<K,V>[] tab = table;;) {
           Node<K,V> f; int n, i, fh;
           // 如果table数组为空或者长度为0(未初始化)，则调用initTable()初始化table
           if (tab == null || (n = tab.length) == 0)
               //通过cas，保证只有一个线程执行初始化过程
               tab = initTable();
           /*
            * 调用实现了CAS原子性操作的tabAt方法
            * tabAt方法的第一个参数是Node数组的引用，第二个参数在Node数组的下标，实现的是在Nod
            * e数组中查找指定下标的Node，如果找到则返回该Node节点(链表头节点)，否则返回null，
            * 这里的i = (n - 1)&hash即是计算待插入的节点在table的下标，即table容量-1的结果和哈
            * 希地址做与运算，和HashMap的算法一样
            */
           else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
               /*
                * 如果该下标上并没有节点(即链表为空)，则直接调用实现了CAS原子性操作的
                * casTable()方法，
                * casTable()方法的第一个参数是Node数组的引用，第二个参数是待操作的下标，第三
                * 个参数是期望值，第四个参数是待操作的Node节点，实现的是将Node数组下标为参数二
                * 的节点替换成参数四的节点，如果期望值和实际值不符返回false，否则参数四的节点成
                * 功替换上去，返回ture，即插入成功。注意这里：如果插入成功了则跳出for循环，插入
                * 失败的话(其他线程抢先插入了)，那么会执行到下面的代码。
                */
               if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                   break;                   // no lock when adding to empty bin
           }
           // 如果tab[i]不为空并且hash值为MOVED，说明该链表正在进行transfer操作，返回扩容完成后的table。
           else if ((fh = f.hash) == MOVED)
               tab = helpTransfer(tab, f);
           else {
               V oldVal = null;
               // 针对首个节点进行加锁操作，而不是segment，进一步减少线程冲突
               synchronized (f) {
                   if (tabAt(tab, i) == f) {
                       //如果该下标上的节点的哈希地址大于等于0，则表示这是个链表
                       if (fh >= 0) {
                           binCount = 1;
                           for (Node<K,V> e = f;; ++binCount) {
                               K ek;
                               // 如果在链表中找到值为key的节点e，直接设置e.val = value即可。
                               if (e.hash == hash &&
                                   ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                   oldVal = e.val;
                                   if (!onlyIfAbsent)
                                       e.val = value;
                                   break;
                               }
                               // 如果没有找到值为key的节点，直接新建Node并加入链表即可。
                               Node<K,V> pred = e;
                               if ((e = e.next) == null) {
                                   pred.next = new Node<K,V>(hash, key,
                                                             value, null);
                                   break;
                               }
                           }
                       }
                       // 如果首节点为TreeBin类型，说明为红黑树结构，执行putTreeVal操作。
                       else if (f instanceof TreeBin) {
                           Node<K,V> p;
                           binCount = 2;
                           //如果插入的结果不为null，则表示为替换
                           if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                          value)) != null) {
                               oldVal = p.val;
                               if (!onlyIfAbsent)
                                   p.val = value;
                           }
                       }
                   }
               }
               if (binCount != 0) {
                   // 如果节点数>＝8，那么转换链表结构为红黑树结构。
                   if (binCount >= TREEIFY_THRESHOLD)
                       treeifyBin(tab, i);
                   if (oldVal != null)
                       return oldVal;
                   break;
               }
           }
       }
       // 计数增加1，有可能触发transfer操作(扩容)。
       addCount(1L, binCount);
       return null;
   }
```
##### 5.2 get方法
- 1、调用spread()方法计算key的hashCode()获得哈希地址；
- 2、计算出键key所在的下标，算法是(n - 1) & h，如果table不为空，且下标上的bucket不为空，则到bucket中查找；
- 3、如果bucket的头节点的哈希地址小于0，则代表这个bucket存储的是红黑树，否则是链表
- 4、到红黑树或者链表中查找，找到则返回该键key的值，找不到则返回null
```java
public V get(Object key) {
       Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
       //计算hashcode
       int h = spread(key.hashCode());
       //判断table是否为空
       if ((tab = table) != null && (n = tab.length) > 0 &&
           (e = tabAt(tab, (n - 1) & h)) != null) {
           //如果哈希地址、键key相同则表示查找到，返回value，这里查找到的是头节点
           if ((eh = e.hash) == h) {
               if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                   return e.val;
           }
           //如果bucket头节点的哈希地址小于0，则代表bucket为红黑树，在红黑树中查找
           else if (eh < 0)
               return (p = e.find(h, key)) != null ? p.val : null;
           //如果bucket头节点的哈希地址不小于0，则代表bucket为链表，遍历链表，在链表中查找
           while ((e = e.next) != null) {
               if (e.hash == h &&
                   ((ek = e.key) == key || (ek != null && key.equals(ek))))
                   return e.val;
           }
       }
       return null;
   }
```
##### 5.3 remove & replaceNode方法
* 1、调用spread()方法计算出键key的哈希地址。
* 2、计算出键key所在的数组下标，如果table为空或者bucket为空，则返回null。
* 3、判断当前table是否正在扩容，如果在扩容则调用helpTransfer方法协助扩容。
* 4、如果table和bucket都不为空，table也不处于在扩容状态，则锁住当前bucket，对bucket进行操作。
* 5、根据bucket的头结点判断bucket是链表还是红黑树。
* 6、在链表或者红黑树中移除哈希地址、键key相同的节点。
* 7、调用addCount方法，将当前table存储的键值对数量-1
```java
public V remove(Object key) {
       return replaceNode(key, null, null);
   }

   /**
    * Implementation for the four public remove/replace methods:
    * Replaces node value with v, conditional upon match of cv if
    * non-null.  If resulting value is null, delete.
    */
   final V replaceNode(Object key, V value, Object cv) {
       //计算需要移除的键key的哈希地址
       int hash = spread(key.hashCode());
       //遍历table
       for (Node<K,V>[] tab = table;;) {
           Node<K,V> f; int n, i, fh;
           //table为空，或者键key所在的bucket为空，则跳出循环返回
           if (tab == null || (n = tab.length) == 0 ||
               (f = tabAt(tab, i = (n - 1) & hash)) == null)
               break;
           //如果当前table正在扩容，则调用helpTransfer方法，去协助扩容
           else if ((fh = f.hash) == MOVED)
               tab = helpTransfer(tab, f);
           else {
               V oldVal = null;
               boolean validated = false;
               //将键key所在的bucket加锁
               synchronized (f) {
                   if (tabAt(tab, i) == f) {
                       if (fh >= 0) {
                           validated = true;
                           for (Node<K,V> e = f, pred = null;;) {
                               K ek;
                               //找到哈希地址、键key相同的节点，进行移除
                               if (e.hash == hash &&
                                   ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                   V ev = e.val;
                                   if (cv == null || cv == ev ||
                                       (ev != null && cv.equals(ev))) {
                                       oldVal = ev;
                                       if (value != null)
                                           e.val = value;
                                       else if (pred != null)
                                           pred.next = e.next;
                                       else
                                           setTabAt(tab, i, e.next);
                                   }
                                   break;
                               }
                               pred = e;
                               if ((e = e.next) == null)
                                   break;
                           }
                       }
                       else if (f instanceof TreeBin) {
                           validated = true;
                           TreeBin<K,V> t = (TreeBin<K,V>)f;
                           TreeNode<K,V> r, p;
                           if ((r = t.root) != null &&
                               (p = r.findTreeNode(hash, key, null)) != null) {
                               V pv = p.val;
                               if (cv == null || cv == pv ||
                                   (pv != null && cv.equals(pv))) {
                                   oldVal = pv;
                                   if (value != null)
                                       p.val = value;
                                   else if (t.removeTreeNode(p))
                                       setTabAt(tab, i, untreeify(t.first));
                               }
                           }
                       }
                   }
               }
               if (validated) {
                   if (oldVal != null) {
                       if (value == null)
                           //调用addCount方法，将当前ConcurrentHashMap存储的键值对数量-1
                           addCount(-1L, -1);
                       return oldVal;
                   }
                   break;
               }
           }
       }
       return null;
   }
```
##### 5.4 initTable方法
- 1、判断table是否为null，即需不需要首次初始化，如果某个线程进到这个方法后，其他线程已经将table初始化好了，那么该线程结束该方法返回。
- 2、如果table为null，进入到while循环，如果sizeCtl小于0(其他线程正在对table初始化)，那么该线程调用Thread.yield()挂起该线程，让出CPU时间，该线程也从运行态转成就绪态，等该线程从就绪态转成运行态的时候，别的线程已经table初始化好了，那么该线程结束while循环，结束初始化方法返回。如果从就绪态转成运行态后，table仍然为null，则继续while循环。
- 3、如果table为null且sizeCtl不小于0，则调用实现CAS原子性操作的compareAndSwap()方法将sizeCtl设置成-1，告诉别的线程我正在初始化table，这样别的线程无法对table进行初始化。如果设置成功，则再次判断table是否为空，不为空则初始化table，容量大小为默认的容量大小(16)，或者为sizeCtl。其中sizeCtl的初始化是在构造函数中进行的，sizeCtl = ((传入的容量大小 + 传入的容量大小无符号右移1位 + 1)的结果向上取最近的2幂次方)
```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        //如果table为null或者长度为0，
        // 则一直循环试图初始化table(如果某一时刻别的线程将table初始化好了，那table不为null，该线程就结束while循环)。
        while ((tab = table) == null || tab.length == 0) {
            /*
             * 如果sizeCtl小于0，即有其他线程正在初始化或者扩容，执行Thread.yield()将当前线程挂起，让出CPU时间，该线程从运行态转成就绪态。
             * 如果该线程从就绪态转成运行态了，此时table可能已被别的线程初始化完成，table不为null，该线程结束while循环
             */
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            /*
             * 如果此时sizeCtl不小于0，即没有别的线程在做table初始化和扩容操作，
             * 那么该线程就会调用Unsafe的CAS操作compareAndSwapInt尝试将sizeCtl的值修改成
             * -1(sizeCtl=-1表示table正在初始化，别的线程如果也进入了initTable方法则会执行
             * Thread.yield()将它的线程挂起 让出CPU时间)，
             * 如果compareAndSwapInt将sizeCtl=-1设置成功 则进入if里面，否则继续while循环
             */
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    //再次确认当前table为null即还未初始化，这个判断不能少
                    if ((tab = table) == null || tab.length == 0) {
                        //如果sc(sizeCtl)大于0，则n=sc，否则n=默认的容量大小16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
##### 5.5 transfer扩容方法
* 1、构建一个nextTable，其大小为原来大小的两倍，这个步骤是在单线程环境下完成的
* 2、将原来table里面的内容复制到nextTable中，这个步骤是允许多线程操作的，所以性能得到提升，减少了扩容的时间消耗
```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        /*
         * 根据服务器CPU数量来决定每个线程负责的bucket数量，避免因为扩容的线程过多反而影响性能。
         * 如果CPU数量为1，则stride=1，否则将需要迁移的bucket数量(table大小)除以CPU数量，平分给
         * 各个线程，但是如果每个线程负责的bucket数量小于限制的最小是(16)的话，则强制给每个线程
         * 分配16个bucket数
         */
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        //如果nextTable还未初始化，则初始化nextTable，这个初始化和iniTable初始化一样，只能由一个线程完成
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        //分配任务和控制当前线程的任务进度，这部分是transfer()的核心逻辑，描述了如何与其他线程协同工作
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```
#### 六、HashMap、HashTable、CurrentHashMap比较

  |   | HashMap  | HashTable |  CurrentHashMap |
  | :------------ |:---------------:| -----:| -----:|
  | 是否线程安全    | 否 | 是 | 是 |
  | 线程安全方法    |  | 采用synchronized类锁,效率低| 采用CAS+synchronized,只锁住当前操作的bucket |
  | 是否允许null    | 是 | 否 | 否 |
  | 数据结构    | 数组＋链表+红黑树 | 数组＋链表 | 数组＋链表+红黑树 |
  | hash算法    | key的hashCode高16位和自已的低16位做异或运算| key.hashcode | (key的hashCode高16位和自已的低16位做异或运算) & 0x7fffffff |
  | 定位方法    | hashcode & (容量大小 - 1) | (hash & 0x7FFFFFFF) % 容量大小| hashcode & (容量大小 - 1) |
  | 扩容方法    | 大于阈值，容量扩充两倍 | newCapacity = (oldCapacity << 1) + 1 | 单线程创建哈希表（容量之前两倍），多线程复制bucket至新表 |
  | 链表插入    | 尾部 | 头部 | 尾部 |
  | 继承的类    | AbstractMap | Dictionary | AbstractMap |
  | 实现接口    | map | map | currentMap |
  | 默认容量    | 16 | 11 | 16 |
  | 默认负载因子    | 0.75 | 0.75 | 0.75 |
  | size方法    | 直接返回成员变量size | 直接返回成员变量count | 遍历CountrCell数组值进行累加，再加baseCount |
