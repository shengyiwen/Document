### ConcurrentHashMap详解

#### 1.类常量

```

    // ConcurrentHashMap允许的最大的容量，这个值必须保持1<<30的大小
    // 因为32bit位的前两个bit位用来做控制的目的
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    
    // 默认的初始化容量，必须是2的平方而且要小于等于MAXIMUM_CAPACITY
    private static final int DEFAULT_CAPACITY = 16;
    
    // 负载因子，在构造器中重写仅影响初始化容量，这个浮点数值通常不会被使用
    // 假话的可以使用{@code n - (n >>> 2)}代替，用来重新计算阈值
    private static final float LOAD_FACTOR = 0.75f;
    
    // 字节Hash编码字段
    
    // 转发节点的hash
    static final int MOVED     = -1;
    
    // 根节点的hash
    static final int TREEBIN   = -2;
    
    // 临时保留的hash
    static final int RESERVED  = -3; 
    
    // 正常节点hash的可用位
    static final int HASH_BITS = 0x7fffffff;

```

#### 2.类方法

```
    // 散列(XOR)较高位的hash值降低并强制高位为0。由于使用二次幂掩码，
    // 因此仅在当前掩码之上的位有所不同的一组has值将始终发生冲突
    // 所以我们应用了一种向下传播高位影响的变换。位扩展的速度、效用和质量之间存在权衡
    // 因为许多常见的哈希集已经合理分布（因此不会从传播中受益）
    // 并且因为我们使用树来处理 bin 中的大量冲突
    // 所以我们只是以最便宜的方式对一些移位的位进行异或以减少系统损失，以及合并最高位的影响，
    // 否则由于表边界而永远不会在索引计算中使用。
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
    
    // 返回给定的容量的2次幂
    // 测试数据，输入，输出 (1, 1), (2, 2), (3, 4), (4, 4), 
    // (5, 8), (6, 8), (16, 16)
    // 得出结论，输入容量，获取最接近输入值且符合2次幂的数
    private static final int tableSizeFor(int c) {
        int n = c - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    
    // 通过Unsafe获取素组的第i个元素
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    // 通过CAS的方式，在坐标为i的地方，存放元素
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

#### 3. 字段

```

    // bin数组，第一次插入时延迟初始化，大小一直是2的幂
    transient volatile Node<K,V>[] table;

    // 下一个要使用的表，仅在调整大小时非空
    private transient volatile Node<K,V>[] nextTable;
    
    // 基本计数器值，主要在没用竞争时使用，但也可以作为表初始化竞争期间的后备，通过CAS更新
    private transient volatile long baseCount;
    
    // 表初始化和调整大小控制
    // 为负数，表示正在初始化或者调整大小，-1表示初始化，否则-(1 + 活动调整大小线程的数量)
    // 当table为null，保存创建时使用的初始表大小，或默认为0
    // 初始化后，保存下一个要调整table大小的元素计数器，即阈值
    private transient volatile int sizeCtl;

```

#### 4. 公共方法

```
    // 创建一个新的，空的map使用默认初始化大容量16
    public ConcurrentHashMap() {
    }
    
    // 创建一个新的map，其初始化table大小可容纳指定的元素数量(initialCapacity)而无需动态调整
    // 而这个时候，table为空，即保存初始化容量
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
    
    // 返回指定键映射到的值，如果此映射中不包含映射的键，返回null
    // 荣誉感以包含此键到值的映射，且存在key.equals，则返回value，否则返回null
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // 计算键的hash值
        int h = spread(key.hashCode());
        // 首先table不能为空，其次长度大于0，且存在此键((n - 1) & h)即可以得到此键所在的idx
        // 然后通过tabAt获取此idx上元素的值Node<K,V>
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            
            // 如果此table的第一个元素的hash值等于指定key的hash，且物理地址一样，或者eqauls，表明找到了，返回此值。否则通过链表next一路比较下去
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
    
    
    // 将指定的键映射到此表中的指定值，键和值都不能为空，可以通过使用与原始键相同的键调用方法来检索该值
    public V put(K key, V value) {
        return putVal(key, value, false);
    }
    
    // 实际的实现，如何存入值
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        // 这里就是为什么ConcurrentHashMap不能存入空值的原因
        if (key == null || value == null) throw new NullPointerException();
        // 获取该键的hash值
        int hash = spread(key.hashCode());
        // 初始化binCount，用于记录相应链表的长度
        int binCount = 0;
        
        // 即将table赋值给tab
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 如果tab为空，则初始化tab，具体初始化话的过程，后面分析
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
                
            // 如果此key所对应的元素是空的，采用CAS存放即可，如果存放成功说明没有并发操作，直接返回
            // 如果存放失败了，说明此时并发了，需进入下一个循环即可
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            
            // 如果此时的hash映射到的头结点元素f的hash为MOVED的时候，表示在扩容，然后帮忙迁移数据，后面分析
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                // 到了这里f即头结点，且f不为空
                V oldVal = null;
                // 获取该数组位置的头节点，然后加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // fh即f的hash值，如果大于0，说明是链表
                        if (fh >= 0) {
                            // 用于累加，记录链表的长度
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果存在equals的key，判断是否进行值覆盖，然后就可以break了
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                // 到了最末端还不存在这个key，然后直接插到最后面
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果是红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            // binCount复制为2
                            binCount = 2;
                            // 使用红黑树的插入方式，插入元素节点
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                
                // binCount不等于0，说明进行了插入操作
                if (binCount != 0) {
                    // 如果binCount大于红黑树转换的临界值，即8
                    if (binCount >= TREEIFY_THRESHOLD)
                        // 这里的处理和HashMap不同，需要table的size也大于64才会转换红黑树，否则进行扩容
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
    
    // 初始化table
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        // 如果table为空，且length = 0
        while ((tab = table) == null || tab.length == 0) {
            // sizeCtl<0表示在初始化或者在扩容，表示目前有其他的线程在处理
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 表示获取到初始化的决定权，然后就修改sizeCtl的值为-1
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // 如果sc大于0，就使用sc的值，否则使用默认容量，参考构造器
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        // 真正初始化的地方
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // sc = 下一次调整table的阈值，也就是0.75 * n
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
    
    // 在给定索引处替换bin中所有的连接节点，除非表太小，在这种情况下改为调整大小
     private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            // 如果table的长度不足64的时候，会进行数组扩容
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            // b是头结点    
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                // 对头结点加锁
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        // 遍历链表，建立红黑树
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        // 将红黑树存放到table映射的位置中
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
    
    // 尝试预先调整table大小满足保存已有的元素
    private final void tryPresize(int size) {
        // 即size是否是>=MAXIMUM_CAPACITY>>>1，如果是使用MAXIMUM_CAPACITY
        // 否则扩容tableSizeFor(size + (size >>> 1) + 1)，上面介绍了tableSizeFor
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            
            // 这里就是初始化table，sc的值就是cap或者默认容量16
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            // 当c小于阈值或者表容量的时候，或n大于最大的容量时，不做任何处理了
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                // 返回用于调整大小为 n 的表的标记位。向左移动 RESIZE_STAMP_SHIFT 时必须为负
                int rs = resizeStamp(n);
                if (sc < 0) {
                    // 在这里nt会使用nextTable，即变量中的nextTable，在扩容时非空，然后将tab的数据，转移到nt中
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
    
   // 帮助迁移数据，其中f为hash到的坐标的头结点
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
    

```