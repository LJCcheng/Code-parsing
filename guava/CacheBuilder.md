## CacheBuilder

创建实例，这个类主要是初始化缓存的一些参数，最大存储容量，最大存储权重，设置访问之后多少时间过期，设置值多久过期等操作，本地缓存的具体操作是在其他类里面进行操作。

```java
 * LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
 *     .maximumSize(10000)
 *     .expireAfterWrite(Duration.ofMinutes(10))
 *     .removalListener(MY_LISTENER)
 *     .build(
 *         new CacheLoader<Key, Graph>() {
 *           public Graph load(Key key) throws AnyException {
 *             return createExpensiveGraph(key);
 *           }
 *         });
```

#### 初始化操作

```java
public CacheBuilder<K, V> concurrencyLevel(int concurrencyLevel) {
    //初始化当前的并发级别
    checkState(
        this.concurrencyLevel == UNSET_INT,
        "concurrency level was already set to %s",
        this.concurrencyLevel);
    checkArgument(concurrencyLevel > 0);
    this.concurrencyLevel = concurrencyLevel;
    return this;
  }

public CacheBuilder<K, V> maximumSize(long maximumSize) {
    //初始化最大存储数量，guava这里存储数量和存储权重只能配置使用一个，如果两个都配置会报错
    checkState(
        this.maximumSize == UNSET_INT, "maximum size was already set to %s", this.maximumSize);
    checkState(
        this.maximumWeight == UNSET_INT,
        "maximum weight was already set to %s",
        this.maximumWeight);
    checkState(this.weigher == null, "maximum size can not be combined with weigher");
    checkArgument(maximumSize >= 0, "maximum size must not be negative");
    this.maximumSize = maximumSize;
    return this;
  }

public CacheBuilder<K, V> maximumWeight(long maximumWeight) {
    //初始化最大存储权重
    checkState(
        this.maximumWeight == UNSET_INT,
        "maximum weight was already set to %s",
        this.maximumWeight);
    checkState(
        this.maximumSize == UNSET_INT, "maximum size was already set to %s", this.maximumSize);
    checkArgument(maximumWeight >= 0, "maximum weight must not be negative");
    this.maximumWeight = maximumWeight;
    return this;
  }

  @CanIgnoreReturnValue
  CacheBuilder<K, V> setValueStrength(Strength strength) {
      //设置值的引用类型
    checkState(valueStrength == null, "Value strength was already set to %s", valueStrength);
    valueStrength = checkNotNull(strength);
    return this;
  }

public CacheBuilder<K, V> expireAfterWrite(long duration, TimeUnit unit) {
    //设置写操作后多久过期
    checkState(
        expireAfterWriteNanos == UNSET_INT,
        "expireAfterWrite was already set to %s ns",
        expireAfterWriteNanos);
    checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
    this.expireAfterWriteNanos = unit.toNanos(duration);
    return this;
  }

public CacheBuilder<K, V> expireAfterAccess(long duration, TimeUnit unit) {
    //设置读操作后多久过期
    checkState(
        expireAfterAccessNanos == UNSET_INT,
        "expireAfterAccess was already set to %s ns",
        expireAfterAccessNanos);
    checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
    this.expireAfterAccessNanos = unit.toNanos(duration);
    return this;
  }

public CacheBuilder<K, V> refreshAfterWrite(long duration, TimeUnit unit) {
    //缓存创建之后多久刷新
    checkNotNull(unit);
    checkState(refreshNanos == UNSET_INT, "refresh was already set to %s ns", refreshNanos);
    checkArgument(duration > 0, "duration must be positive: %s %s", duration, unit);
    this.refreshNanos = unit.toNanos(duration);
    return this;
  }

public <K1 extends K, V1 extends V> CacheBuilder<K1, V1> removalListener(
      RemovalListener<? super K1, ? super V1> listener) {
    //增加移除key的监听器
    checkState(this.removalListener == null);

    // safely limiting the kinds of caches this can produce
    @SuppressWarnings("unchecked")
    CacheBuilder<K1, V1> me = (CacheBuilder<K1, V1>) this;
    me.removalListener = checkNotNull(listener);
    return me;
  }

public <K1 extends K, V1 extends V> LoadingCache<K1, V1> build(
      CacheLoader<? super K1, V1> loader) {
    //创建本地缓存
    checkWeightWithWeigher();
    //这个 LocalLoadingCache 是继承了LocalManualCache类
    return new LocalCache.LocalLoadingCache<>(this, loader);
  }
```

#### LocalCache类

这个类就是guavaCache实现的关键，它继承了AbstractMap，并且实现了ConcurrentMap接口，和concurrentHashMap结构类似，但是内部没有红黑树的优化，只有数组和链表

```java
LocalCache(
      CacheBuilder<? super K, ? super V> builder, @CheckForNull CacheLoader<? super K, V> loader) {
    //并发级别
    concurrencyLevel = Math.min(builder.getConcurrencyLevel(), MAX_SEGMENTS);

    //key引用和value引用
    keyStrength = builder.getKeyStrength();
    valueStrength = builder.getValueStrength();

    //key和value的比较器
    keyEquivalence = builder.getKeyEquivalence();
    valueEquivalence = builder.getValueEquivalence();

    //最大权重，当前权重，访问后过期时间，写入后过期时间，创建后的刷新时间
    maxWeight = builder.getMaximumWeight();
    weigher = builder.getWeigher();
    expireAfterAccessNanos = builder.getExpireAfterAccessNanos();
    expireAfterWriteNanos = builder.getExpireAfterWriteNanos();
    refreshNanos = builder.getRefreshNanos();

    //缓存移除监听器，唤醒队列
    removalListener = builder.getRemovalListener();
    removalNotificationQueue =
        (removalListener == NullListener.INSTANCE)
            ? LocalCache.discardingQueue()
            : new ConcurrentLinkedQueue<>();

    //获取时间，key和value枚举工厂，操作计数器，不存在缓存默认执行方法
    ticker = builder.getTicker(recordsTime());
    entryFactory = EntryFactory.getFactory(keyStrength, usesAccessEntries(), usesWriteEntries());
    globalStatsCounter = builder.getStatsCounterSupplier().get();
    defaultLoader = loader;

    //初始化容量
    int initialCapacity = Math.min(builder.getInitialCapacity(), MAXIMUM_CAPACITY);
    if (evictsBySize() && !customWeigher()) {
      initialCapacity = (int) Math.min(initialCapacity, maxWeight);
    }

    // Find the lowest power-of-two segmentCount that exceeds concurrencyLevel, unless
    // maximumSize/Weight is specified in which case ensure that each segment gets at least 10
    // entries. The special casing for size-based eviction is only necessary because that eviction
    // happens per segment instead of globally, so too many segments compared to the maximum size
    // will result in random eviction behavior.
    int segmentShift = 0;
    int segmentCount = 1;
    while (segmentCount < concurrencyLevel && (!evictsBySize() || segmentCount * 20 <= maxWeight)) {
      ++segmentShift;
      segmentCount <<= 1;
    }
    this.segmentShift = 32 - segmentShift;
    segmentMask = segmentCount - 1;

    //创建分段锁
    this.segments = newSegmentArray(segmentCount);

    //每一个锁的初始容量
    int segmentCapacity = initialCapacity / segmentCount;
    if (segmentCapacity * segmentCount < initialCapacity) {
      ++segmentCapacity;
    }

    //计算最接近初始容量的2的次方，因为concurrentHashMap 是两倍扩容
    int segmentSize = 1;
    while (segmentSize < segmentCapacity) {
      segmentSize <<= 1;
    }

    if (evictsBySize()) {
        //按权重
      // Ensure sum of segment max weights = overall max weights
      long maxSegmentWeight = maxWeight / segmentCount + 1;
      long remainder = maxWeight % segmentCount;
      for (int i = 0; i < this.segments.length; ++i) {
        if (i == remainder) {
          maxSegmentWeight--;
        }
          //为每个分段添加元素
        this.segments[i] =
            createSegment(segmentSize, maxSegmentWeight, builder.getStatsCounterSupplier().get());
      }
    } else {
        //按数量
      for (int i = 0; i < this.segments.length; ++i) {
          //为每个分段添加元素
        this.segments[i] =
            createSegment(segmentSize, UNSET_INT, builder.getStatsCounterSupplier().get());
      }
    }
  }
```



#### get()方法

这里是使用的`LocalManualCache`类的get()方法，它实现了`cache`接口，这个`cache`接口自己定义了一些`map`的常用方法，比如`put,get,getIfPresent`等，因为guavaCache是使用的`concurrentHashMap`，这个接口应该是为了适配，所以定义了map的一些方法。

```java
public V get(K key, final Callable<? extends V> valueLoader) throws ExecutionException {
      checkNotNull(valueLoader);
      //执行函数式接口方法
      return localCache.get(
          key,
          new CacheLoader<Object, V>() {
            @Override
            public V load(Object key) throws Exception {
              return valueLoader.call();
            }
          });
    }

V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {
    int hash = hash(checkNotNull(key));
    //获取分段，再从分段中获取元素
    return segmentFor(hash).get(key, hash, loader);
  }

V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
      checkNotNull(key);
      checkNotNull(loader);
      try {
        if (count != 0) { // read-volatile
          // don't call getLiveEntry, which would ignore loading values
            //查找到的元素
          ReferenceEntry<K, V> e = getEntry(key, hash);
          if (e != null) {
            long now = map.ticker.read();
              //key,value为null为会返回null，过期了也会返回null
              //这里如果key，value为null，会触发移除监听器执行移除方法
            V value = getLiveValue(e, now);
            if (value != null) {
                //不为空，如果设置有缓存刷新时间，并且满足刷新的条件，就开异步任务刷新
              recordRead(e, now);
              statsCounter.recordHits(1);
              return scheduleRefresh(e, key, hash, value, now, loader);
            }
            ValueReference<K, V> valueReference = e.getValueReference();
            if (valueReference.isLoading()) {
                //value为空，从引用中获取，这里的valueReference为value的引用类型，强引用弱引用等
                //从引用中获取
              return waitForLoadingValue(e, key, valueReference);
            }
          }
        }

        // at this point e is either null or expired;
          //如果为null或者过期，重新加载
        return lockedGetOrLoad(key, hash, loader);
      } catch (ExecutionException ee) {
        Throwable cause = ee.getCause();
        if (cause instanceof Error) {
          throw new ExecutionError((Error) cause);
        } else if (cause instanceof RuntimeException) {
          throw new UncheckedExecutionException(cause);
        }
        throw ee;
      } finally {
          //清空writeQueue和accessQueue元素以及弱引用
        postReadCleanup();
      }
    }


V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
      ReferenceEntry<K, V> e;
      ValueReference<K, V> valueReference = null;
      LoadingValueReference<K, V> loadingValueReference = null;
      boolean createNewEntry = true;

      lock();
      try {
        // re-read ticker once inside the lock
        long now = map.ticker.read();
          //清理弱引用和过期的writeQueue和accessQueue
        preWriteCleanup(now);

        int newCount = this.count - 1;
        AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
        int index = hash & (table.length() - 1);
        ReferenceEntry<K, V> first = table.get(index);

        //遍历链表
        for (e = first; e != null; e = e.getNext()) {
          K entryKey = e.getKey();
          if (e.getHash() == hash
              && entryKey != null
              && map.keyEquivalence.equivalent(key, entryKey)) {
            valueReference = e.getValueReference();
            if (valueReference.isLoading()) {
                //是否创建新引用
              createNewEntry = false;
            } else {
              V value = valueReference.get();
              if (value == null) {
                  //value为null
                enqueueNotification(
                    entryKey, hash, value, valueReference.getWeight(), RemovalCause.COLLECTED);
              } else if (map.isExpired(e, now)) {
                  //value过期
                // This is a duplicate check, as preWriteCleanup already purged expired
                // entries, but let's accommodate an incorrect expiration queue.
                enqueueNotification(
                    entryKey, hash, value, valueReference.getWeight(), RemovalCause.EXPIRED);
              } else {
                  //直接返回
                recordLockedRead(e, now);
                statsCounter.recordHits(1);
                // we were concurrent with loading; don't consider refresh
                return value;
              }

              // immediately reuse invalid entries
                //移除不合法的值
              writeQueue.remove(e);
              accessQueue.remove(e);
              this.count = newCount; // write-volatile
            }
            break;
          }
        }

        if (createNewEntry) {
            //创建新引用
          loadingValueReference = new LoadingValueReference<>();

          if (e == null) {
            e = newEntry(key, hash, first);
            e.setValueReference(loadingValueReference);
            table.set(index, e);
          } else {
            e.setValueReference(loadingValueReference);
          }
        }
      } finally {
        unlock();
        postWriteCleanup();
      }

      if (createNewEntry) {
        try {
          // Synchronizes on the entry to allow failing fast when a recursive load is
          // detected. This may be circumvented when an entry is copied, but will fail fast most
          // of the time.
          synchronized (e) {
              //执行loader加载方法
            return loadSync(key, hash, loadingValueReference, loader);
          }
        } finally {
          statsCounter.recordMisses(1);
        }
      } else {
        // The entry already exists. Wait for loading.
        return waitForLoadingValue(e, key, valueReference);
      }
    }
```

#### put()方法

```java
public V put(K key, V value) {
    checkNotNull(key);
    checkNotNull(value);
    int hash = hash(key);
    return segmentFor(hash).put(key, hash, value, false);
  }

V put(K key, int hash, V value, boolean onlyIfAbsent) {
      //分段锁，锁住当前segment
      lock();
      try {
        long now = map.ticker.read();
          //移除弱引用
        preWriteCleanup(now);

        int newCount = this.count + 1;
        if (newCount > this.threshold) { // ensure capacity
            //扩容，和hashmap和concurrentHashMap差不多，两倍扩容，0.75扩容因子，遍历链表
          expand();
          newCount = this.count + 1;
        }

        AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
        int index = hash & (table.length() - 1);
        ReferenceEntry<K, V> first = table.get(index);

        // Look for an existing entry.
        for (ReferenceEntry<K, V> e = first; e != null; e = e.getNext()) {
          K entryKey = e.getKey();
          if (e.getHash() == hash
              && entryKey != null
              && map.keyEquivalence.equivalent(key, entryKey)) {
            // We found an existing entry.
              //链表已经存在对应节点

            ValueReference<K, V> valueReference = e.getValueReference();
            V entryValue = valueReference.get();

            if (entryValue == null) {
              ++modCount;
              if (valueReference.isActive()) {
                enqueueNotification(
                    key, hash, entryValue, valueReference.getWeight(), RemovalCause.COLLECTED);
                setValue(e, key, value, now);
                newCount = this.count; // count remains unchanged
              } else {
                setValue(e, key, value, now);
                newCount = this.count + 1;
              }
              this.count = newCount; // write-volatile
              evictEntries(e);
              return null;
            } else if (onlyIfAbsent) {
              // Mimic
              // "if (!map.containsKey(key)) ...
              // else return map.get(key);
                //设置访问时间以及往accessQueue添加元素
              recordLockedRead(e, now);
              return entryValue;
            } else {
              // clobber existing entry, count remains unchanged
                //替换
              ++modCount;
              enqueueNotification(
                  key, hash, entryValue, valueReference.getWeight(), RemovalCause.REPLACED);
              setValue(e, key, value, now);
              evictEntries(e);
              return entryValue;
            }
          }
        }

          //创建
        // Create a new entry.
        ++modCount;
        ReferenceEntry<K, V> newEntry = newEntry(key, hash, first);
        setValue(newEntry, key, value, now);
        table.set(index, newEntry);
        newCount = this.count + 1;
        this.count = newCount; // write-volatile
        evictEntries(newEntry);
        return null;
      } finally {
        unlock();
        postWriteCleanup();
      }
    }
```

