### 单例模式的代码：

    // 第一种实现方式 
    public class Singleton {
        private static Singleton instance = null;
        
        public synchronized static Singleton getInstance() {
            if (instance == null) {
                instance = new Singleton();
            }
            return instance;
        }
    }

    // 第二种实现方式，性能最好
    public class Singleton {
        private static Singletion instance = new Singleton();
        
        public static Singleton getInstance() {
        return instance;
        }
    } 

    //第三种实现方式
    public class Singleton {
        private static Singleton instance = null;
        public static Singleton getInstance() {
        if (instance == null) {
            synchronized(Singletion.class) {
                if (instance == null) {
                    instance = new Singleton();
                    }
                }
            }
            return instance;
        }
    }


#### 为什么第三种模式要通过双重锁模世

- 因为方法锁远没有代码块锁性能好，当存在的时候，直接返回了实例对象，不存在锁竞争问题
  
- 如果不存在，线程1进入锁区域，线程2进行锁等待
  
- 如果没有里面的判断。线程1进入线程后，new一个对象走了，然后释放锁，线程2进入，没有判断，则会直接创建一个对象，
  这个时候，就存在了两个实例对象！而在锁内部增加if判断，当线程1 new一个对象走了以后，线程2进入，这个时候判断发现，有对象了，然后就不会new了

#### 通过双重锁模式的好处

- 锁代码块的性能更好
  
- 锁中的代码最多只会被执行两次

#### 双重锁的风险

但是使用双重锁模式以然存在风险，风险点在于，java内存模型中，除了基础类型的赋值是原子性的，其他的都是分多个步骤

初始化的过程分为如下步骤: 

- 申请一片内存空间

- 初始化变量

- 将对象句柄指向内存空间

如果释放锁的时候，只完成了上述步骤的仅前两个的话，就会出现线程2拿到的对象是一个空的句柄，这个时候使用的时候就会出现空指针的问题

#### 如何避免：

使用volatile关键字，为什么volatile可以避免，因为volatile关键字，会是修饰的变量对所有的线程可见，
即一个线程发生改变的时候，其他的线程获取的都是最新的值，而且volatile是禁止指令重排的。
volatile发生修改的时候，会强制刷新主存里的数据，然后使高速缓存的值失效，
因此所有的线程读取的是最新的变量值。而且volatile 保证了禁止指令重拍。
因此volatile只有可见性、有序性、但是不具备原子性，要保证原子性要和synchronized关键字一起使用。

详细可以参考[Java内存模型](./Java内存模型.md)
