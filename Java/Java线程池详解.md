### 为什么使用线程池

- 线程是稀缺资源，无限制分配线程，会消耗大量的系统资源甚至导致系统崩溃，因此为了合理的使用线程以及进行监控可以使用线程池来解决

- 多线程可以减少处理器的闲置时间，提高系统的吞吐能力

### 使用线程池的好处

- 降低资源消耗: 降低线程创建和销毁造成的消耗

- 提高响应速度: 任务到来不需要重新启动线程

- 提高线程的可管理性等

### Java线程池的分类

- CachedThreadPool

可缓存的线程池，根据需要创建新的线程，但是可以用时将重用之前重用的线程池，这些池通常会提高执行许多短期异步任务的程序性能，如果没有可用的线程，则会创建新的线程并添加到线程池中，60秒内未使用的线程将会被终止并从池中删除。核心线程数为0，最大线程数为Integer.Max_VALUE

```
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

- ScheduleThreadPool

周期性执行任务的线程池，适用于执行周期性的任务，有核心线程，但也有非核心线程，非核心线程的大小也可以无限大

```
public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }
```

- SingleThreadPool

只有一个线程用来执行任务，适用于有顺序任务的应用场景

```
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

- FixedThreadPool

固定的线程池，有核心线程，核心线程就是最大线程，没有非核心线程

```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

### 线程池的执行流程

![线程池执行流程](../images/thread_pool_execute_sequence.png)

### 类的关系

- 1. Executor接口

```java
public interface Executor {
    // 只定义了execute接口，用于执行线程    
    void execute(Runnable command);
}
```

- 2. ExecutorService接口
    
```java
public interface ExecutorService extends Executor {
    
    // 启动有序关闭，其中执行先前提交的任务，不接受新的任务
    // 如果已经关闭，调用没有其他的影响
    // 该方法不会的等待之前提交的任务完成执行
    void shutdown();
    
    // 尝试停止所有正在执行的任务，停止等待任务的处理，并返回等待执行的任务列表
    // 此方法不会等待主动执行的任务终止，除了尽力尝试停止处理正在执行的任务之外，没有任何保证
    List<Runnable> shutdownNow();
    
    // 判断当前执行器是否已经停止
    boolean isShutdown();

    // 如果关闭后所有的任务都已经完成，则返回true
    // 请注意，除非之前调用了shutdown或shutdownNow，否则永远不会返回true
    boolean isTerminated();

    // 阻塞直到所有的任务在关闭请求后完成执行，或发生超时，或当前线程被中断，以先发生为准
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    // 提交一个返回值任务以供执行，并发挥一个标识任务未决结果的Future
    <T> Future<T> submit(Callable<T> task);

    // 提交一个任务以供执行，并返回一个代表该任务的Future。 Future的Get方法将在成功完成后返回给定的结果。
    <T> Future<T> submit(Runnable task, T result);
    
    // 提交一个任务以供执行，Future的get方法将会返回null知道成功执行完成
    Future<?> submit(Runnable task);
    
    // 批量提交任务执行，在所有任务完成时保存他们的状态和结果，Future的isDone方法对于返回列表的每个元素都是true
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

   
    // 执行给定任务，当全部完成或超时到期时，返回一个保存结果和状态的Future列表，以先发生者为准
    // 返回时，如果没有完成的任务会被取消
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    // 执行给定的任务，返回成功完成的任务的结果（即不抛出异常），如果有的话。在正常或异常返回时，未完成的任务将被取消  
    // 如果在此操作进行时修改了给定的集合，则此方法的结果未定义。
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    // 同上
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

- 3. AbstractExecutorService类
    
这个类封装了一些通用方法等，主要的实现在execute(Runnable command)中，此方法由子类实现

- 4. ThreadPoolExecutor类
    
```java

public class ThreadPoolExecutor extends AbstractExecutorService {

    // 主线程控制状态ctl是一个原子整数，封装了两个概念字段
    // 1.workCount，表示有效的线程数
    // 2.runState，表示是否运行，关闭等
    // 为了将他们打包到一个int值，我们将workerCount先知道了2^19-1个线程，而不是2^31-1个，将来可能AtomicLong代替
    //
    // workerCount是允许启动和不允许停止的工人数量，该值可能与实际活动线程数暂时不同，例如当ThreadFactory在被询问
    // 时未创建线程时，以及退出线程在终止前仍在执行，用户可见的池大小报告为工作集的当前大小
    //
    // runStat提供主要的生命周期控制
    // RUNNING：接受新的任务并处理排队任务
    // SHUTDOWN：不接受新的任务，单处理排队任务
    // STOP: 不接受新的任务，不处理派对任务，并中断正在进行的任务
    // TIDYING: 所有的任务都已终止，workCount为0，切换到任务TIDYING的线程运行terminate()钩子方法
    // TERMINATED: terminate()已完成
    
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // 运行状态
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 获取运行状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 获取worker数量
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    
    // 控制workerCount数量
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }

    // 用于保存任务和移交给工作线程的队列，不要求workQueue.poll()返回空就意味着workerQueue.isEmpty()
    // 所以只能依赖isEmpty来检查队列是否为空（决定是否从SHUTDOWN转到TIDYING时必须这样做）
    // 这是用于特殊用途的队列，例如允许poll()返回Null的DelayQueues，即使它稍后可能再延迟到期时返回null
    private final BlockingQueue<Runnable> workQueue;

    private final ReentrantLock mainLock = new ReentrantLock();

    // 包含池中所有工作线程的集合。仅在持有 mainLock 时访问。
    private final HashSet<Worker> workers = new HashSet<Worker>();

    // 等待条件支持 awaitTermination
    private final Condition termination = mainLock.newCondition();

    // 跟踪达到的最大池大小。只能在 mainLock 下访问。
    private int largestPoolSize;

    // 完成任务的计数器。仅在工作线程终止时更新。只能在 mainLock 下访问。
    private long completedTaskCount;

    private volatile ThreadFactory threadFactory;

    // 拒绝策略
    private volatile RejectedExecutionHandler handler;

    // 等待工作的空闲线程的超时（以纳秒为单位）。当存在超过 corePoolSize 或允许 CoreThreadTimeOut 时
    // 线程使用此超时。否则他们永远等待新的工作。
    private volatile long keepAliveTime;

    private volatile boolean allowCoreThreadTimeOut;

    // 核心线程数和最大线程数
    private volatile int corePoolSize;
    private volatile int maximumPoolSize;
    
    // 默认的拒绝策略
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();


    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");

    /* The context to be used when executing the finalizer, or null. */
    private final AccessControlContext acc;

    /**
     * Class Worker mainly maintains interrupt control state for
     * threads running tasks, along with other minor bookkeeping.
     * This class opportunistically extends AbstractQueuedSynchronizer
     * to simplify acquiring and releasing a lock surrounding each
     * task execution.  This protects against interrupts that are
     * intended to wake up a worker thread waiting for a task from
     * instead interrupting a task being run.  We implement a simple
     * non-reentrant mutual exclusion lock rather than use
     * ReentrantLock because we do not want worker tasks to be able to
     * reacquire the lock when they invoke pool control methods like
     * setCorePoolSize.  Additionally, to suppress interrupts until
     * the thread actually starts running tasks, we initialize lock
     * state to a negative value, and clear it upon start (in
     * runWorker).
     */
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        // 1. 如果运行的线程数少于corePoolSize，请尝试使用给定命令启动一个新线程作为其第一个任务,
        // 对addWorker的调用以原子方式检查runState和workerCount,
        // 从而通过返回 false 来防止在不应该添加线程时出现误报
        // 2.如果任务可以成功排队，那么我们仍然需要仔细检查是否应该添加一个线程
        //（因为自上次检查以来现有线程已死亡）或池自进入此方法后关闭。
        // 因此，我们重新检查状态，并在必要时在停止时回滚入队，如果没有则启动一个新线程。
        // 3.如果我们不能排队任务，那么我们尝试添加一个新线程。如果它失败了，
        //我们知道我们已经关闭或饱和，因此拒绝该任务
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (!isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                    firstTask == null &&
                    ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
}

```