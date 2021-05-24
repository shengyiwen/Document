### 1.强引用

类似于Object a = new Object()，仅当a被置为null的时候，此对象才会被gc给回收掉，我们的代码中大部分是此类代码。只要强引用存在，垃圾回收器将永远不会回收此对象，哪怕内存不足，JVM会直接抛出OutOfMemory。

### 2.软引用

    SoftReference<Student> soft = new SoftReference<>();

当内存不足，会触发GC，**如果GC后内存仍不足**，就会把软引用当对象回收。即GC后内存仍不足才会被回收，所以软引用多数用于缓存方面。

### 3.弱引用

WeakReference<Student> weak = new WeakReference();

当发生GC，无论内存是否充足都会被回收。就是说只要发生GC，这类弱引用对象就在劫难逃，**无论内存是否足够，只要JVM开始了垃圾回收，这些弱引用关联的对象都会被回收。**

### 4.虚引用

    ReferenceQueue queue = new ReferenceQueue();
    
    PhantomReference<Student> phantom = new PhantomReference<>(queue);

虚引用无法通过引用获取一个对象的真实引用，但是被被回收的时候，会把当前引用加入道queue中。
因此使用在Nio管理堆外内存。如果一个对象仅持有虚引用，那么和没有任何引用一样，随时都有可能被回收。
此对象的get方法返回的一直为null，因此无法获取值，虚引用必须和ReferenceQueue引用队列一起使用。

    public class PhantomReference<T> extends Reference<T> {
        publci T get() {
            return null;
        }
        public PhantomReference(T referent, ReferenceQueue<T super T> q) {
            super(referent, q);
        }
    }


**引用队列**

引用队列可以和软引用、弱引用、虚引用一同使用，当垃圾回收器准备回收一个对象时，如果发现它还在被引用，那么就会在回收这个对象之前放入到与之关联的引用队列中去。程序可以通过判断应用队列中是否已经加入了引用，来判断此对象是否将要被垃圾回收，这样就可以在对象被回收之前采取一些必要的措施

### 引用的强度

强引用 > 软引用 > 弱引用 > 虚引用
