## CountDownLatch源码

这个类一般用于多线程的任务计数，每个线程完成任务之后，可以执行这个类的内部方法进行监控，内部有个暂停的方法，任务全都完成之后，可以继续执行后续的逻辑。

它内部还是通过内部类继承`AQS`，来实现主要代码逻辑。

```java
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            //调用AQS设置state
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            //只有当state为0才为1
            return (getState() == 0) ? 1 : -1;
        }

    	//请求共享变量
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                //获取state
                int c = getState();
                //state为0代表共享变量为0，意思就是已经没有资源可供获取
                if (c == 0)
                    return false;
                int nextc = c-1;
                //CAS操作进行替换
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```

#### 构造方法

```java
public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
    	//创建sync内部类
        this.sync = new Sync(count);
    }
```

#### await()方法

```java
public void await() throws InterruptedException {
    	//一直等待
        sync.acquireSharedInterruptibly(1);
    }

public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
    	//调用AQS方法，tryAcquireShared方法只有state为0的时候才大于0，这里小于0证明有资源可供获取
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    	//这个方法是将当前节点放入等待链表
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            //死循环
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    //大于0证明state为0，任务执行完毕
                    if (r >= 0) {
                        //唤醒等待线程，有一个等待链表
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                //获取失败资源失败判断是否需要挂起，以及是否需要中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

#### await(long timeout, TimeUnit unit)方法

```java
public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
    	//设置超时等待，这里逻辑和await差不多，就是在AQS里面加了一个超时的功能
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }
```

#### countDown()方法

```java
public void countDown() {
        sync.releaseShared(1);
    }

public final boolean releaseShared(int arg) {
    	//调用CAS比较替换，cas减一成功并且state为0才返回true
        if (tryReleaseShared(arg)) {
            //唤醒等待线程
            doReleaseShared();
            return true;
        }
        return false;
    }

private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                //获取node节点状态
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    //节点状态为signal表示后续节点需要被唤醒
                    //cas将signal替换为0
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    //唤醒线程
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }

private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
    	//cas将node节点小于0的替换为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            //倒序遍历，将node节点状态小于等于0的节点赋值给s节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //唤醒s节点
            LockSupport.unpark(s.thread);
    }
```

