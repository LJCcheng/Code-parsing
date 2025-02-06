## `ThreadPoolExecutor`线程池源码

简介：`ThreadPoolExecutor`继承自抽象类`AbstractExecutorService`，抽象类又实现了接口`Executor`，总体设计类似于模板设计模式，抽象了一些公共方法

### 重要属性和状态

```java
	// 工作线程数和队列元素的总和
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	// Integer.size 大小为32
    private static final int COUNT_BITS = Integer.SIZE - 3;
	// 2的29次方 - 1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
	//-1 向左移动29位，1110 0000 0000 0000 0000 0000 0000 0000
    private static final int RUNNING    = -1 << COUNT_BITS;
	// 0 向左移动29位 0000 0000 0000 0000 0000 0000 0000 0000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
	// 1 向左移动29位 0010 0000 0000 0000 0000 0000 0000 0000
    private static final int STOP       =  1 << COUNT_BITS;
	// 2 向左移动29位 0100 0000 0000 0000 0000 0000 0000 0000
    private static final int TIDYING    =  2 << COUNT_BITS;
	// 3 向左移动29位 0110 0000 0000 0000 0000 0000 0000 0000
    private static final int TERMINATED =  3 << COUNT_BITS;

	// 阻塞队列
	private final BlockingQueue<Runnable> workQueue;

    /**
     * Lock held on access to workers set and related bookkeeping.
     * While we could use a concurrent set of some sort, it turns out
     * to be generally preferable to use a lock. Among the reasons is
     * that this serializes interruptIdleWorkers, which avoids
     * unnecessary interrupt storms, especially during shutdown.
     * Otherwise exiting threads would concurrently interrupt those
     * that have not yet interrupted. It also simplifies some of the
     * associated statistics bookkeeping of largestPoolSize etc. We
     * also hold mainLock on shutdown and shutdownNow, for the sake of
     * ensuring workers set is stable while separately checking
     * permission to interrupt and actually interrupting.
     */
	// 锁
    private final ReentrantLock mainLock = new ReentrantLock();

    /**
     * Set containing all worker threads in pool. Accessed only when
     * holding mainLock.
     */
	// 保存工作任务，方便后续统计
    private final HashSet<Worker> workers = new HashSet<Worker>();

    /**
     * Wait condition to support awaitTermination
     */
    private final Condition termination = mainLock.newCondition();

    /**
     * Tracks largest attained pool size. Accessed only under
     * mainLock.
     */
	// 最大工作线程峰值
    private int largestPoolSize;

    /**
     * Counter for completed tasks. Updated only on termination of
     * worker threads. Accessed only under mainLock.
     */
	// 已完成任务数量
    private long completedTaskCount;

    /*
     * All user control parameters are declared as volatiles so that
     * ongoing actions are based on freshest values, but without need
     * for locking, since no internal invariants depend on them
     * changing synchronously with respect to other actions.
     */

    /**
     * Factory for new threads. All threads are created using this
     * factory (via method addWorker).  All callers must be prepared
     * for addWorker to fail, which may reflect a system or user's
     * policy limiting the number of threads.  Even though it is not
     * treated as an error, failure to create threads may result in
     * new tasks being rejected or existing ones remaining stuck in
     * the queue.
     *
     * We go further and preserve pool invariants even in the face of
     * errors such as OutOfMemoryError, that might be thrown while
     * trying to create threads.  Such errors are rather common due to
     * the need to allocate a native stack in Thread.start, and users
     * will want to perform clean pool shutdown to clean up.  There
     * will likely be enough memory available for the cleanup code to
     * complete without encountering yet another OutOfMemoryError.
     */
	// 线程工厂
    private volatile ThreadFactory threadFactory;

    /**
     * Handler called when saturated or shutdown in execute.
     */
	// 拒绝策略
    private volatile RejectedExecutionHandler handler;

    /**
     * Timeout in nanoseconds for idle threads waiting for work.
     * Threads use this timeout when there are more than corePoolSize
     * present or if allowCoreThreadTimeOut. Otherwise they wait
     * forever for new work.
     */
	// 线程存活时间
    private volatile long keepAliveTime;

    /**
     * If false (default), core threads stay alive even when idle.
     * If true, core threads use keepAliveTime to time out waiting
     * for work.
     */
	// 是否允许线程超时
    private volatile boolean allowCoreThreadTimeOut;

    /**
     * Core pool size is the minimum number of workers to keep alive
     * (and not allow to time out etc) unless allowCoreThreadTimeOut
     * is set, in which case the minimum is zero.
     */
	// 核心线程数
    private volatile int corePoolSize;

    /**
     * Maximum pool size. Note that the actual maximum is internally
     * bounded by CAPACITY.
     */
	// 最大容量
    private volatile int maximumPoolSize;

    /**
     * The default rejected execution handler
     */
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
```



### 判断状态和获取任务个数

```java
	// CAPACITY 为2的29次方-1，~取反后，为1110 0000 0000 0000 0000 0000 0000 0000，然后与当前任务数进行与运算
	private static int runStateOf(int c)     { return c & ~CAPACITY; }
	// 不取反直接与任务个数进行与运算
    private static int workerCountOf(int c)  { return c & CAPACITY; }
	// 进行或运算
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```



### 核心类 Worker

简介：它实现了 Runnable 接口，里面保存了已经完成的任务个数和任务以及线程，这里面有线程复用的逻辑，它还继承了AQS，方便执行run方法的时候加锁和释放锁

```java
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
            // 如果当前worker的状态为cancel就中断线程
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

final void runWorker(Worker w) {
    	// 获取当前线程，获取当前任务
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 这里就是线程复用的方法，while循环实现，这里getTask方法就是任务源
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 对当前线程池的状态，当前线程的状态进行判断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    //前置方法
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //后置方法
                        afterExecute(task, thrown);
                    }
                } finally {
                    //执行完毕之后置空，方便gc
                    task = null;
                    //当前worker完成任务数
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //结束操作
            processWorkerExit(w, completedAbruptly);
        }
    }

private void processWorkerExit(Worker w, boolean completedAbruptly) {
    	// 这里的入参completedAbruptly只有try-catch代码块抛异常的时候才会为true
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            // cas将工作线程数目移除
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 统计完成的任务数
            completedTaskCount += w.completedTasks;
            // 移除worker
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
    	// 线程池的运行状态为running或者shutdown
        if (runStateLessThan(c, STOP)) {
            // 前面代码块没有抛异常为false
            if (!completedAbruptly) {
                //如果允许核心线程存活超时，找到最小值
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    //最小为0的时候，如果队列不为空，赋值1，表示至少需要一个
                    min = 1;
                // 如果工作线程数大于最小值就返回
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            //添加到worker集合中，任务传null，这里应该是为了快速完成其他任务才添加的新worker
            addWorker(null, false);
        }
    }
```



### execute方法

```java
public void execute(Runnable command) {
    	// 任务不能为null
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
    	// 获取工作线程数以及线程池状态
        int c = ctl.get();
    	// 如果工作线程数小于核心线程数
        if (workerCountOf(c) < corePoolSize) {
            // 就添加核心线程，添加成功就结束，添加失败就继续执行下面逻辑
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
    	// 线程池状态为running，往等待队列中添加任务
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 二次校验
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果工作线程数（核心线程为0的情况），就添加一个线程，但是任务为null
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
    	//如果等待队列已满，入队失败的情况，就添加非核心线程，如果为核心线程添加失败，就执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```





#### `addWorker`方法

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            //获取线程池的运行状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //如果状态不为运行状态，判断等待队列中的任务是否为空
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                //获取工作线程数
                int wc = workerCountOf(c);
                //如果工作线程数超过限制就不能添加
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //工作线程自增，自增成功就结束
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //线程池运行状态已经改变，cas操作，重新执行
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //新建worker（工作线程）
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    //获取运行状态
                    int rs = runStateOf(ctl.get());

                    //running或者shutdown状态传的null任务
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        //重新设置最大线程数峰值
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //调用worker的run方法执行runworker方法
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //执行失败
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```



##### addWorkerFailed方法

线程池执行失败的时候执行

```java
private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                //移除工作线程
                workers.remove(w);
            //移除工作线程数量
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }

final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            //判断线程池状态是running，或者小于tidying，或者为shutdown并且阻塞队列不为空
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //cas替换线程池状态
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //这个是状态变为tidying之后执行的方法，现目前是空方法，需要自己重写
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        //调用condition的signalAll方法，唤醒等待线程池终止的线程
                        termination.signalAll();	
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```



### 阻塞队列 `workQueue`

简介：它是线程池中将任务保存起来的队列，然后在线程复用的时候将任务从队列中取出来使用

```java
private final BlockingQueue<Runnable> workQueue;
```

`BlockingQueue`接口下的子类代表了不同类型的阻塞队列，拥有不同的特性

```java
ArrayBlockingQueue -> 有界队列，内部结构数组
LinkedBlockingQueue -> 无界队列，内部结构链表
PriorityBlockQueue -> 优先级无界队列
DelayedWorkQueue -> 延迟无界队列
```



### 拒绝策略

```java
AbortPolicy -> 直接拒绝
CallerRunsPolicy -> 线程池不是shutdown状态，直接用当前线程直接执行
DiscardOldestPolicy -> 丢弃最老的任务
DiscardPolicy -> 什么都不做
```



#### AbortPolicy

直接拒绝

```java
public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```



#### CallerRunsPolicy

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                //用当前线程执行
                r.run();
            }
        }
    }
```



#### DiscardOldestPolicy

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                //线程池不是shutdown状态的情况下，丢弃最老的任务，然后把这个任务放入队列
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```



#### DiscardPolicy

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            //什么都不做
        }
    }
```



