#### ReentrantReadWriteLock源码

`ReentrantReadWriteLock`源码和 `ReentrantLock`源码有很多相似地方，笔记主要记录读写锁的实现，与`ReentrantLock`相同部分去看`ReentrantLock`笔记。

```java
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock; //读锁
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;//写锁
    /** Performs all synchronization mechanics */
    final Sync sync;//和ReentrantLock的实现相似，读写锁功能的实现都在里面
```

读写锁也分了公平锁和不公平锁，笔记不做描述（去看`ReentrantLock`笔记）。

```java
public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```

##### Sync抽象内部类

```java
abstract static class Sync extends AbstractQueuedSynchronizer {

        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }

        protected final boolean tryAcquire(int acquires) {
            
        }

        protected final boolean tryReleaseShared(int unused) {
            
        }

        private IllegalMonitorStateException unmatchedUnlockException() {
            return new IllegalMonitorStateException(
                "attempt to unlock read lock, not locked by current thread");
        }

        protected final int tryAcquireShared(int unused) {
            
        }

        /**
         * Full version of acquire for reads, that handles CAS misses
         * and reentrant reads not dealt with in tryAcquireShared.
         */
        final int fullTryAcquireShared(Thread current) {
            
        }

        /**
         * Performs tryLock for write, enabling barging in both modes.
         * This is identical in effect to tryAcquire except for lack
         * of calls to writerShouldBlock.
         */
        final boolean tryWriteLock() {
            
        }

        /**
         * Performs tryLock for read, enabling barging in both modes.
         * This is identical in effect to tryAcquireShared except for
         * lack of calls to readerShouldBlock.
         */
        final boolean tryReadLock() {
            
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final Thread getOwner() {
            // Must read state before owner to ensure memory consistency
            return ((exclusiveCount(getState()) == 0) ?
                    null :
                    getExclusiveOwnerThread());
        }

        final int getReadLockCount() {
            return sharedCount(getState());
        }

        final boolean isWriteLocked() {
            return exclusiveCount(getState()) != 0;
        }

        final int getWriteHoldCount() {
            return isHeldExclusively() ? exclusiveCount(getState()) : 0;
        }

        final int getReadHoldCount() {
            if (getReadLockCount() == 0)
                return 0;

            Thread current = Thread.currentThread();
            if (firstReader == current)
                return firstReaderHoldCount;

            HoldCounter rh = cachedHoldCounter;
            if (rh != null && rh.tid == getThreadId(current))
                return rh.count;

            int count = readHolds.get().count;
            if (count == 0) readHolds.remove();
            return count;
        }

        final int getCount() { return getState(); }
    }
```

##### 写锁

调用`Sync`内部方法，将写操作封装到写锁

```java
public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;

        /**
         * Constructor for use by subclasses
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
    
        public void lock() {
            //加锁
            sync.acquire(1);
        }
    
        public boolean tryLock( ) {
            //尝试获取写锁
            return sync.tryWriteLock();
        
        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            //尝试获取锁，超时时间timeout
            return sync.tryAcquireNanos(1, unit.toNanos(timeout));
        }
     
        public void unlock() {
            //释放锁
            sync.release(1);
        }
    
        public boolean isHeldByCurrentThread() {
            //当前线程是否持有锁
            return sync.isHeldExclusively();
        }
       
        public int getHoldCount() {
            //获取写锁数量
            return sync.getWriteHoldCount();
        }
    }
```

###### `lock`()方法

调用`AbstractQueuedSynchronizer`父类的`acquire`方法

```java
public final void acquire(int arg) {
    	//尝试获取锁，并且线程加入等待队列，acquireQueued()方法看ReentrantLock笔记
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

protected final boolean tryAcquire(int acquires) {
           
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                //有线程已经获取到锁
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //没有线程获取写锁 或者 当前线程不是持有锁的线程，结束
                //因为获取写锁需要等待读锁完毕才能获取
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //获取写锁的数量大于最大限制报错
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                //设置state
                setState(c + acquires);
                return true;
            }
    		//没有线程获取到锁，先判断当前线程是否需要阻塞，这里就是公平锁和非公平锁的区别
    		//公平锁这里需要判断该线程之前有没有线程在等待获取到锁，获取不到就会返回true
    		//非公平锁就不用判断直接返回false
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

###### `tryLock`()方法

```java
public boolean tryLock( ) {
            return sync.tryWriteLock();
        }

final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                //获取当前获取写锁的线程数量
                int w = exclusiveCount(c);
                //读写互斥，需要等待读锁释放才能获取锁
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }

public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
    		//超时时间获取
            return sync.tryAcquireNanos(1, unit.toNanos(timeout));
        }

public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
    	//尝试获取锁，超时时间内持续获取锁
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

//整体和ReentrantLock差不多，不多记录
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
    	//超时时间小于等于0
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            //写循环获取
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                //超时时间到了，结束
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

##### 读锁

调用`Sync`内部方法，将读操作封装为读锁

```java
public static class ReadLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -5992448646407690164L;
        private final Sync sync;

        /**
         * Constructor for use by subclasses
         *
         * @param lock the outer lock object
         * @throws NullPointerException if the lock is null
         */
        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        public void lock() {
            sync.acquireShared(1);
        }


        public boolean tryLock() {
            return sync.tryReadLock();
        }
    
        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            //和写锁差不多
            return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
        }

    }
```

###### `lock`()方法

```java
public final void acquireShared(int arg) {
    	//尝试获取锁
        if (tryAcquireShared(arg) < 0)
            //获取不到锁，死循环获取
            doAcquireShared(arg);
    }

protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
    		//存在写锁和获取锁的线程不是当前线程
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
    		//获取获取读锁的数量
            int r = sharedCount(c);
    		//判断获取读锁是否需要阻塞，这里获取读锁需要阻塞主要是公平锁和非公平锁的区别
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    //初始化
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    //第一个获取读锁
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    //读锁加1
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }

private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    //头结点获取锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //获取到锁结束
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //获取锁失败判断是否可以阻塞和中断（看ReentrantLock笔记）
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
    		//死循环尝试获取锁
            for (;;) {
                int c = getState();
                //获取写锁数量以及判断当前线程是否持有写锁
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    //判断读锁是否需要阻塞
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            //初始化
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                //超出最大限制
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        //初始化
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        //当前线程加1
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        //读锁加1
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

###### `tryLock`()方法

```java
final boolean tryReadLock() {
            Thread current = Thread.currentThread();
    		//死循环
            for (;;) {
                int c = getState();
                //读写互斥
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                    return false;
                int r = sharedCount(c);
                //最大限制
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }
```



