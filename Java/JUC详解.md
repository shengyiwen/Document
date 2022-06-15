### 1. 可重入锁

同一个线程在最外部获取到锁，在进入到该线程内部的方法会自动获取锁，前提是这个锁对象是同一个，不会因为之前没有释放锁而获取不到锁，可以很大程度上避免死锁

### 2. LockSupport的解释

用于创建锁和其他同步类的基本线程阻塞原语。可以理解为线程wait/notify的升级版。有park和unpark两个方法用来阻塞线程和解除阻塞

#### 三种让线程等待和阻塞的方法

- Object的wait和notify方法
- JUC包的Condition的await和signal方法
- LockSupport类的park和unpark方法

#### LockSupport

LockSupport底层是一个通过Unsafe类的实现的，通过permit进行控制，permit默认是0，所以刚开始调用park方法，当前线程阻塞，直到别的线程将当前线程的permit设置为1时，park方法被唤醒，然后会将permit再次设置为0并返回。
permit不会累加的，即可以理解为只能为0和1。如果当前有凭证的时候，会消费凭证正常退出等待，如果没有凭证就会进入等待，而且最多仅能有1个permit

原码实现:

    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
    
    public static void park() {
        UNSAFE.park(false, 0L);
    }

测试用例:

    @Test
    public void lockSupportParkAndUnPark() throws InterruptedException {
        Thread threadA = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread a come in");
                LockSupport.park();
                System.out.println("thread a come out");
            }
        });
        threadA.start();

        Thread.sleep(3000L);

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread b come in");
                LockSupport.unpark(threadA);
                System.out.println("thread b notify");
            }
        }).start();
    }

LockSupport的和Object以及Condition的区别在于，Object和Condition的wait必须在notify之前，不然就会一直进入阻塞。但是LockSupport可以先唤醒后等待，这样的话，线程进入park方法的时候就不会进入阻塞


### 3. AQS的实现类

AQ是构建锁或者其他同步器组件重量级的基础框架以及整个JUC的基石。通过一个int类型的state和双向的链表做的Queue构建而成。详细可以看AQS详解

#### CycleBarrier使用详解

可以用于达到栅栏的时候做什么操作，可以异步跑批

```java
public class CycliceBarrierTest {
    
    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(1000, new Runnable() {
        @Override
        public void run() {
            System.out.println("到达了栅栏的阀值");
        }
    });
    
    
    public static void main(String[] args) {
        for (int i = 0; i < 1000; i ++) {
            int finalI = i;
            new Thread(() -> {
                System.out.println("i am ready. i:" + finalI);
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
    
}
```

#### Semaphore用法

同时允许多少线程通过，其他的等待

```java
public class SemaphoreTest {

        public static void main(String[] args) {
            Semaphore semaphore = new Semaphore(3);
    
            for (int i = 0; i < 10; i ++) {
                int finalI = i;
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            semaphore.acquire();
                            System.out.println("i have acquire semaphore. i:" + finalI);
                            Thread.sleep(2000L);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } finally {
                            semaphore.release();
                        }
    
                    }
                }).start();
            }
            
        }
    }
```