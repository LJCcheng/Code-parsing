## AtomicInteger源码

#### 内部属性

功能的主要实现就是使用Unsafe类，调用它的CAS方法来实现

```java
	// setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            //将value通过unsafe类映射到valueOffset属性上，这里意思就是获取AtomicInteger类的value的内存地址,
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

#### get()方法

```java
public final int get() {
        return value;
    }
```

#### set()方法

```java
public final void set(int newValue) {
        value = newValue;
    }
```

#### getAndSet()方法

先获取原来值再设置新值

```java
public final int getAndSet(int newValue) {
    	//调用unsafe的getAndSetInt方法
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            //这里表示获取当前值，如果cas一直失败这里就一直获取，直到成功
            var5 = this.getIntVolatile(var1, var2);
            
            //CAS替换
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }
```

#### getAndIncrement()方法

获取原值然后加1

```java
public final int getAndIncrement() {
    	//这里传this是为了先获取对象的内存地址，然后通过valueOffset偏移量来获取value的内存地址，以此来操作value的值
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            //自旋获取
            var5 = this.getIntVolatile(var1, var2);
            //cas替换
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

#### getAndDecrement()方法

获取原值，然后减1

```java
public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
```

#### getAndAccumulate()方法

获取原值，然后调用函数式接口计算新值进行cas操作

```java
public final int getAndAccumulate(int x,
                                      IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            //调用get方法
            prev = get();
            //函数式接口计算新值
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSet(prev, next));
        return prev;
    }
```

#### getAndUpdate()方法

获取原值，然后调用函数式接口在原值的基础上进行计算，最后cas替换

```java
public final int getAndUpdate(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            //调用get方法
            prev = get();
            //函数式接口计算新值
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return prev;
    }
```

