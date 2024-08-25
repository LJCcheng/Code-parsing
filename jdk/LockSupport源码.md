## LockSupport源码

这个类就是AQS里面挂起线程的具体操作

#### park()方法

```java
public static void park(Object blocker) {
   		//获取当前线程
        Thread t = Thread.currentThread();
    	//调用Unsafe类的putObject方法设置parkBlockerOffset
        setBlocker(t, blocker);
    	//线程永久阻塞
        UNSAFE.park(false, 0L);
    	//阻塞结束之后重置parkBlockerOffset，如果不重置，在获取值的时候获取到的是前一次的值，脏数据
        setBlocker(t, null);
    }

private static void setBlocker(Thread t, Object arg) {
        // Even though volatile, hotspot doesn't need a write barrier here.
    	//内存偏移量设置值
        UNSAFE.putObject(t, parkBlockerOffset, arg);
    }
```

#### parkNanos()方法

```java
public static void parkNanos(Object blocker, long nanos) {
    	//超时时间大于0才执行
        if (nanos > 0) {
            Thread t = Thread.currentThread();
            setBlocker(t, blocker);
            //设置超时时间，时间到了之后就会被唤醒
            UNSAFE.park(false, nanos);
            setBlocker(t, null);
        }
    }
```

#### parkUntil()方法 

```java
public static void parkUntil(Object blocker, long deadline) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
    	//第一个参数为true，表示设置有关联唤醒信号，当信号被唤醒的时候或者超时时间到了之后线程才回唤醒
        UNSAFE.park(true, deadline);
        setBlocker(t, null);
    }
```

