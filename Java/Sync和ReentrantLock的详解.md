了解java并发的应该都会知道如果要在java中实现同步只能通过synchronized或者Lock来实现，但是为什么我们使用synchronized就能实现，还需要Lock呢，因此我们需要详细的了解synchronized和Lock的区别和功能

### 1. synchronized的缺陷:

- 使用synchronized的时候，如果一个线程被阻塞的时候。没有超时处理，其他的线程会一直陷入阻塞队列中
释放锁的情况：执行完代码块  代码块抛出异常

- 读写锁问题，使用synchronized的时候，无论是读与写，还是读与读都会发生锁现象

- synchronized不能知道当前的是否是上锁状态

而Lock可以解决上面的问题

### 2. Lock介绍:

Lock是一个接口，提供如下方法

    public interface Lock {
        void lock();
        void lockInterruptibly() throws InterruptedException;
        boolean tryLock();
        boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
        void unlock();
        Condition newCondition();
    }

- lock()方法是获取锁，如果当前锁已经被占用，则需要等待。且Lock加锁，需要用户自己进行手动释放，否则就会进入死锁

- tryLock()方法是尝试获取锁，如果获取到锁就返回true，否则返回false

- tryLock(long time, TimeUnit unit)则是在有限时间内获取锁，如果获取到返回true，如果超时返回false

- lockInterruptibly()方法则是当此线程在获取此锁的时候，如果陷入等待状态，这个时候调用这个线程的interrupt()能够中断此线程的等待状态。当一个线程已经获取锁的话是不能被中断的。synchronized也是没法被中断的，只能一直等下去

**ReentrantLock "可重入锁”**， 实现了Lock接口的实现，使用时候要注意：ReentrantLock作为一个锁，不能作为局部变量，作为局部变量不能起到作用

ReadWriteLock 读写锁是一个接口

    public interface ReadWriteLock {
        /**
         * Returns the lock used for reading.
         *
         * @return the lock used for reading.
         */
        Lock readLock();
     
        /**
         * Returns the lock used for writing.
         *
         * @return the lock used for writing.
         */
        Lock writeLock();
    }

一个用来获取读锁，一个用来获取写锁。也就是说将文件的读写操作分开，分成2个锁来分配给线程，从而使得多个线程可以同时进行读操作。下面的ReentrantReadWriteLock实现了ReadWriteLock接口

ReentrantReadWriteLock

ReentrantReadWriteLock里面提供了很多丰富的方法，不过最主要的有两个方法：**readLock()**和**writeLock()**用来获取读锁和写锁

### 3.Lock和synchronized的选择

总结来说，Lock和synchronized有以下几点不同：

- Lock是一个接口，而synchronized是Java中的关键字，synchronized是内置的语言实现


- synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁


- Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断


- 通过Lock可以知道有没有成功获取锁，而synchronized却无法办到

- Lock可以提高多个线程进行读操作的效率。在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时（即有大量线程同时竞争），此时Lock的性能要远远优于synchronized。所以说，在具体使用时要根据适当情况选择


1.可重入锁

如果锁具备可重入性，则称作为可重入锁。像synchronized和ReentrantLock都是可重入锁，可重入性在我看来实际上表明了锁的分配机制：基于线程的分配，而不是基于方法调用的分配。举个简单的例子，当一个线程执行到某个synchronized方法时，比如说method1，而在method1中会调用另外一个synchronized方法method2，此时线程不必重新去申请锁，而是可以直接执行方法method2。而由于synchronized和Lock都具备可重入性，所以不会发生上述现象

2.可中断锁

可中断锁：顾名思义，就是可以相应中断的锁。在Java中，synchronized就不是可中断锁，而Lock是可中断锁。如果某一线程A正在执行锁中的代码，另一线程B正在等待获取该锁，可能由于等待时间过长，线程B不想等待了，想先处理其他事情，我们可以让它中断自己或者在别的线程中中断它，这种就是可中断锁。在前面演示lockInterruptibly()的用法时已经体现了Lock的可中断性

3.公平锁

公平锁即尽量以请求锁的顺序来获取锁。比如同是有多个线程在等待一个锁，当这个锁被释放时，等待时间最久的线程（最先请求的线程）会获得该所，这种就是公平锁。非公平锁即无法保证锁的获取是按照请求锁的顺序进行的。这样就可能导致某个或者一些线程永远获取不到锁。在Java中，synchronized就是非公平锁，它无法保证等待的线程获取锁的顺序。而对于ReentrantLock和ReentrantReadWriteLock，它默认情况下是非公平锁，但是可以设置为公平锁

在ReentrantLock中定义了2个静态内部类，一个是NotFairSync，一个是FairSync，分别用来实现非公平锁和公平锁。我们可以在创建ReentrantLock对象时，通过以下方式来设置锁的公平性：

    ReentrantLock lock = new ReentrantLock(true);

4.读写锁

读写锁将对一个资源（比如文件）的访问分成了2个锁，一个读锁和一个写锁。正因为有了读写锁，才使得多个线程之间的读操作不会发生冲突。ReadWriteLock就是读写锁，它是一个接口，ReentrantReadWriteLock实现了这个接口。可以通过readLock()获取读锁，通过writeLock()获取写锁
