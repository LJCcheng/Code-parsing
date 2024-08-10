## ReentrantLock源码

`ReentrantLock` 实现了`lock`接口，重写其中方法，但是主要实现在 `Sync`抽象静态内部类中，相当于 `ReentrantLock` 起到包装的作用。这里的 `Sync`是抽象类，有两个子类，分别是 `FairSync`公平锁和 `NonfairSync`非公平锁。

#### Sync抽象类

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
    	//子类去实现
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            //获取当前加锁状态，为0代表未加锁
            if (c == 0) {
                //cas替换
                if (compareAndSetState(0, acquires)) {
                    //设置持有锁的线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                //如果当前线程持有锁，重新设置state，相当于可重入锁
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            //计算释放锁后的state
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                //state为0代表没有线程获取锁，重置获取锁的线程
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    }
```

#### `FairSync`公平锁

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            //加锁
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //这里和非公平锁的区别是这里调用hasQueuedPredecessors()方法判断了是否有其他线程在等待锁
                //如果有其他线程在等待锁，就加锁失败
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
    	//1.头结点不等于尾结点
    	//2.头结点的下一节点为null 或者 头结点的下一节点的所属线程不是当前线程 => 返回true
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

#### `NonfairSync`非公平锁

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            //调用sync父类方法
            return nonfairTryAcquire(acquires);
        }
    }
```

#### `AbstractQueuedSynchronizer`抽象类

它是 `Sync`类的父类

##### `acquire`()方法

```java
public final void acquire(int arg) {
    	//获取不到锁，并且线程可中断
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //调用线程中断方法intercept()
            selfInterrupt();
    }
```

##### `acquireQueued`()方法

```java
final boolean acquireQueued(final Node node, int arg) {
    	//是否加锁失败
        boolean failed = true;
        try {
            //线程是否可以中断
            boolean interrupted = false;
            //死循环
            for (;;) {
                //获取当前节点前一节点
                final Node p = node.predecessor();
                //如果是头结点，并且获取到锁
                if (p == head && tryAcquire(arg)) {
                    //将当前节点设置为头结点
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //获取锁失败阻塞线程并且线程被中断了
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //这里faild只有报错的情况才会为true
            if (failed)
                cancelAcquire(node);
        }
    }
```

##### `shouldParkAfterFailedAcquire`()方法

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
    	//前一节点为阻塞状态
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            //前一节点是取消状态
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            //跳过取消状态的节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            //不为阻塞和取消状态，将前一节点设为阻塞状态
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

##### `parkAndCheckInterrupt`()方法

```java
private final boolean parkAndCheckInterrupt() {
    	//调用Unsafe类的方法来阻塞线程
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

##### `cancelAcquire`()方法

```java
private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;

        // Skip cancelled predecessors
        Node pred = node.prev;
    `	//跳过取消状态的节点
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
    
    	//重新获取节点
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;

        // If we are the tail, remove ourselves.
        if (node == tail && compareAndSetTail(node, pred)) {
            //尾结点就清除
            compareAndSetNext(pred, predNext, null);
        } else {
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                //重新设置下一节点
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //唤醒其他线程
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```

##### `unparkSuccessor`()

```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
    	//跳过取消状态线程
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
    	//调用unpark唤醒线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```



