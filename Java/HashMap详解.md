### HashMap涉及到的内容

#### 1.默认大小、负载因子以及扩容倍数

- 默认初始化容量为16，默认负载因子是0.75
- threshold = 数组长度 *,负载因子，当元素个数大于等于threshold时，HashMap会进行自动扩容    
- table数组中存放指向链表的引用，这里table数组不是在构造方法里面初始化的，而是resize()里进行扩容的


**为什么初始化容量是16，负载因子是0.75呢？**

我们都知道HashMap数组长度被设计成为2的幂次方，那为什么初始容量不是4、8、或者32呢，其实Jdk设计者权衡后的一个比较合理的数据。如果默认容量是8的话，当元素为6的时候，就要进行扩容，扩容非常浪费内存和cpu的，32的话如果只添加少量元素则会浪费内存，因此设计成为16比较合适，负载因子也是同理

#### 2.底层数据结构

HashMap在Jdk1.8中的实现是由数组+链表+红黑树实现，Jdk1.7中是由数组+链表实现

下面是HashMap的一些属性字段

    // 初始容量是1 << 4，即16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4
    // 负载因子是0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75f
    // Hash数组，在resize中初始化
    transient Node<K,V>[] table
    // 元素个数
    transient int size
    // 容量阀值，元素个数大于这个数的时候进行自动扩容
    int threshold

下面是对table数组中Node对象的介绍，Node是HashMap的一个内部类，用来表示一个key-value，原码如下

    static class Node<K,V> implement Map.Entry<K,V> {
        final int hash;
        final K key, V value;
        Node<K,V> next;
        
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        
        public final int hashCode() {
            // ^ 相同返回0，不同返回1
            return Object.hashCode(key) ^ Objects.hashCode(value);
        }
        
        public final boolean equals(Object o) {
            if (o == this) {
                return true;
            }
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey) && Objects.equals(value, e.getValue())) {
                    return true;
                }
            }
            return false;
        }
    }

#### 3.如何处理hash冲突

查考下面6的参入

#### 4.如何计算key的hash值

key值的计算原码

    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

这里的hash计算，让高位与低位进行异或运算，变相的让高位数据参与计算，init有32位，右移16位就能让低16位和高16位进行异或，这样为了增加hash值的随机性

#### 5.数组长度为什么是2的幂次方

HashMap中通过一个tableSizeFor的方法来确保HashMap数组长度永远为2的幂次方，原码如下

    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n > MAXIMUM_CAPACITY) ? MAXIMUN_CAPACITY : n + 1;
    }

不考虑最大容量大情况下，是返回大于等于输入参数且最近的2的整数次幂的数，该算法就是让最高位的1后面的位都变为1，然后让结果+1，然后得到2的整数次幂

什么时候调用到tableSizeFor呢，即在构造方法里用该方法设置threshold，也就是容量阈值

    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0) {
            throw new IllegalArgumentException();
        }
        if (initialCapacity > MAXIMUN_CAPACITY) {
            throw new IllegalArgumentException();
        }
        if (loadFactor <= 0 || Float.isNal(loadFactor)) {
            throw new IllegalArgumentException();
        }
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

**为什么把数组的长度设计为2的幂次方呢？**

- 当数组长度为2的幂次方时，可以使用位运算计算元素在数组中的下标

HashMap是通过index = hash&(table.length - 1)这个公式来计算在table数组中存放的下标，就是把元素的Hash值为数组长度-1的值做一个与运算，这个公司等价于hash%length，但是前提是数组长度是2的幂次方的时候，这个等价条件才有效

- 增加hash值的随机性，减少hash碰撞

如果length是2的幂次方，则lenght-1转化为二次方必定是1111...的形式，这样的话，所有位置都能与元素值进行与运算，如果不是2的次幂，则length-1最后一位是0，则运算的结果最后一位永远为0，浪费空间


#### 6.查找、插入、扩容过程

**扩容**

HashMap每次扩容都是建立一个新的table数组，长度和阈值都变为原来的两倍，然后把原数组元素重新映射到新的数组上，具体如下：

- 首先判断table数组长度，如果大于0说明已经初始化过了，那么按照当前table数组长度进行2倍扩容，阈值也变为2倍

- 若table数组未被初始化过，则阈值大于0，说明调用了HashMap方法，那么就把数组大小设置成为threshlod

- 若table数组未被初始化，且threshold为0，就调用默认初始化方法，数组大小为16，阈值threshold = 16 * 0.75

- 如果不是第一次初始化，那么扩容以后，需要重新计算键值对位置，并把他们映射到合适的位置上去，如果及诶但是红黑树类型的话，则需要进行红黑数的拆分

需要注意的是JDK1.8以后的扩容映射不需要重新像1.7那样每个都要重新计算了，而是通过hash&oldCap的值来判断，若为0则索引位置不变，不为0则新索引为:原索引+老的数组长度。具体原因就是因为使用了2次幂的扩展，所以元素的位置要么是原来位置，要么就是再移动2次幂个位置。

**链表树化**

指的是将链表转为红黑树，转为红黑树有两个条件

- 链表长度大于8

- table数组长度大于等于64

当table容量比较小的时候，键值对hash碰撞的概率比较高，进而导致链表长度比较长，这个时候应该优先扩容，而不是树化

**红黑树拆分**

就是扩容后对元素进行重新映射，红黑树可能会被拆分为两条链表

**查找**

- 通过key计算hash,找到在数组中的位置
- 判断上面是否有节点，如果为null，则返回
- 判断该节点是否是要查找的元素，如果是的则返回
- 如果不是，则判断是节点类型，如果是红黑树，则调用红黑树的方法进行查找
- 如果是链表，则遍历调用equals方法进行计算

**插入**

- 当table为空的时候，通过扩容的方式初始化table

- 通过计算key的hash值，求出下标，若该位置没有元素，则创建新的Node节点

- 若发生了hash碰撞，遍历链表查询key是否存在，如果存在则替换

- 如果不存在，则将元素查到链表的尾部，并根据链表长度决定是否转换为
红黑树

- 判断键值对数量是否大于等于阈值，如果是的话，则进行扩容。

JDK1.8的时候是插入队尾，JDK1.7的时候插入队头，因为1.7版本认为新插入的数据，使用频率会更高，但是头插法在多线程环境下会导致两个节点相互引用，形成死循环

#### 7.fail-fast机制

在进行forEach的时候进行remove操作，抛出ConcurrentModificationException。这个就是我们常说的fail-fast快速失败机制

这里我们从一个变量说起，即 transient int modCount;

这个变量，表示集合被改变的次数，修改指定是插入或者删除操作，每次插入和删除的时候，modCount都会进行自增。当我们进行遍历的时候，每次遍历下个元素的时候，都会被modeCount进行判断，如果发现不一致则表示修改，就会发生异常。
