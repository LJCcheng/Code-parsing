## ThreadLocal源码

#### 初始化initialValue()

threadlocal初始化方法，初始化属性

```java
protected T initialValue() {
        return null;
    }

public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }
```

#### `SuppliedThreadLocal`子类

它是threadLocal的静态内部类，通过函数式接口提供初始化值，只起到了这个作用

```java
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    	//函数式接口
        private final Supplier<? extends T> supplier;

        SuppliedThreadLocal(Supplier<? extends T> supplier) {
            this.supplier = Objects.requireNonNull(supplier);
        }

        @Override
        protected T initialValue() {
            return supplier.get();
        }
    }
```

#### `get`方法

```java
public T get() {
    	//获取当前线程
        Thread t = Thread.currentThread();
    	//获取当前线程的ThreadLocalMap，这里ThreadLocalMap存在每个线程之中，ThreadLocal的主要实现就在里面
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //调用ThreadLocalMap的getEntry方法获取值
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
    	//表示还没初始化
        return setInitialValue();
    }

ThreadLocalMap getMap(Thread t) {
    	//获取当前线程的threadLocal
        return t.threadLocals;
    }

private Entry getEntry(ThreadLocal<?> key) {
    		//位运算获取table的下标
            int i = key.threadLocalHashCode & (table.length - 1);
    		//获取元素
            Entry e = table[i];
    		//找到元素
            if (e != null && e.get() == key)
                return e;
            else
                //没有找到元素（hash冲突）或者entry为null
                return getEntryAfterMiss(key, i, e);
        }

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

    		//这里相当于是在解决hash冲突
            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    //清除下标为i的元素，防止内存泄漏（主动清理）
                    expungeStaleEntry(i);
                else
                    //查找下一个元素，就是下标加1
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
    		//剪枝，将下标为staleSlot的元素值清空，防止内存泄漏
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
    		//查找下一元素
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                //这里将ThreadLocal为null的值清空，因为key为弱引用，gc的时候就会被回收，但是value是强引用，不会被回收，就一直占用内存，这里主动清理，防止内存泄漏
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    //这里i和h不相等的话以h为准
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        //这里是处理tab[h]=null的情况，清除为null
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        //找到tab[h]为null的情况，重新赋值
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

#### setInitialValue方法

```java
private T setInitialValue() {
    	//获取初始化值，上面提到过
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
    	//存在设值，不存在新建
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
        if (this instanceof TerminatingThreadLocal) {
            TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
        }
        return value;
    }

void createMap(Thread t, T firstValue) {
    	//这里会new 一个ThreadLocalMap来保存线程变量
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

#### isPresent()方法

```java
boolean isPresent() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
    	//判断是否为null
        return map != null && map.getEntry(this) != null;
    }
```

#### set()方法

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
    }
```

#### 重头戏ThreadLocalMap

ThreadlLocalMap在每次调用get，set方法的时候都会主动清除key为null的数据，这里是为了防止内存泄漏。

##### 重要属性

```java
private static final int INITIAL_CAPACITY = 16; //默认容量

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;//存储每个线程的变量

private int threshold; // Default to 0 扩容因子
```

##### 这里为什么是Entry数组？

因为一个线程可以有多个`threadLocal`，这里Entry的下标是通过threadLocal 的 threadLocalhashCode 进行的与操作。



`entry`继承自`WeakReference`，`entry`中的`referent`在gc的时候就会被回收，但是value被 `entry`强引用，就不会回收

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

##### set()方法

```java
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
    		//获取key的下标位置
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                //获取引用
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    //找到引用，覆盖
                    e.value = value;
                    return;
                }

                if (k == null) {
                    //引用为null，替换
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

    		//entry为null，创建
            tab[i] = new Entry(key, value);
            int sz = ++size;
    		//清除槽位，size大于扩容因子就重新扩容
    		//cleanSomeSlots就是循环调用expungeStaleEntry方法，前面已经介绍
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

##### rehash()方法

```java
private void rehash() {
    		//清除key为null的元素
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
    		//size达到扩容因子的3/4，和hashMap一样
            if (size >= threshold - threshold / 4)
                resize();
        }

private void resize() {
    		//2倍扩容，循环遍历将旧的table转移到新的table
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```

##### replaceStaleEntry()方法

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    //从后往前，找到为null的值
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
    		//从前往后遍历	
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    //清除为null的引用
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
              	//清除为null的引用
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```



#### 疑问解答

1. 既然正常情况下线程在每次使用之后都会销毁，那么线程里面保存的`threadlocalMap`也会被销毁，那就不存在内存泄漏了；但是如果在使用线程池的情况下，线程复用了之后，线程里面的`threadlocal`是不会销毁的，但是在每次访问`threadlocal`的时候，它会清理`entry[]`数组里面`threadlocal`引用为`null`的值，这里应该没有网上说的那么严重才对。

   ```txt
   1）每次访问threadlocal的时候，它是不会每次都清理entry[]数组为null的threadlocal，只会在get方法获取不到的时候才会清理，以及set方法设置的key不存在的时候等才会清理，并不是每次都会清理。
   2）内存泄漏和内存溢出概念不同，内存泄漏是threadlocal被回收，导致无法通过threadlocal访问这个资源，导致这个资源一直存在在内存，无法被回收，无法被再次访问。
   ```

   
