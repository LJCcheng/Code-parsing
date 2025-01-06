## CacheFilter

简介：这个`cacheFilter`是`dubbo`自己封装的缓存存储容器，包含了`threadlocal`，`lru`，`lfu`，`jcache`，`expiring`等内存淘汰策略

### `CacheFactory`工厂

```java
@SPI("lru")
public interface CacheFactory {
    @Adaptive({"cache"})
    Cache getCache(URL url, Invocation invocation);
}
```

### `Cache`接口

```java
public interface Cache {
    void put(Object key, Object value);

    Object get(Object key);
}
```

由工厂创建`Cache`实例，常见的设计模式

### `AbstractCacheFactory`抽象工厂

```java
public abstract class AbstractCacheFactory implements CacheFactory {
    private final ConcurrentMap<String, Cache> caches = new ConcurrentHashMap();
    private final Object MONITOR = new Object();

    public AbstractCacheFactory() {
    }

    public Cache getCache(URL url, Invocation invocation) {
        //获取url上的一些参数，组装为key
        url = url.addParameter("method", invocation.getMethodName());
        String key = url.getServiceKey() + invocation.getMethodName();
        //获取缓存，这里有caches 线程安全的concurrentHashMap存储限流实例
        Cache cache = (Cache)this.caches.get(key);
        if (null != cache) {
            return cache;
        } else {
            //加锁
            synchronized(this.MONITOR) {
                cache = (Cache)this.caches.get(key);
                if (null != cache) {
                    return cache;
                } else {
                    //创建限流实例，模板设计，实现交由子类，抽取公共逻辑
                    cache = this.createCache(url);
                    this.caches.put(key, cache);
                    return cache;
                }
            }
        }
    }

    protected abstract Cache createCache(URL url);
}
```

### `threadLocal` 线程共享缓存

#### `threadlocalCacheFactory `具体实例工厂

```java
public class ThreadLocalCacheFactory extends AbstractCacheFactory {
    public ThreadLocalCacheFactory() {
    }

    protected Cache createCache(URL url) {
        return new ThreadLocalCache(url);
    }
}
```

#### `threadlocalCache`实例

`threadlocal` 源码笔记已详细介绍~~~

```java
public class ThreadLocalCache implements Cache {
    private final ThreadLocal<Map<Object, Object>> store = ThreadLocal.withInitial(HashMap::new);

    public ThreadLocalCache(URL url) {
    }

    public void put(Object key, Object value) {
        ((Map)this.store.get()).put(key, value);
    }

    public Object get(Object key) {
        return ((Map)this.store.get()).get(key);
    }
}
```

### `lru`实例

#### LruCacheFactory 具体实例工厂

```java
public class LruCacheFactory extends AbstractCacheFactory {
    public LruCacheFactory() {
    }

    protected Cache createCache(URL url) {
        return new LruCache(url);
    }
}
```

#### lruCache 实例

最少使用

```java
public class LruCache implements Cache {
    private final Map<Object, Object> store;

    public LruCache(URL url) {
        int max = url.getParameter("cache.size", 1000);
        this.store = new LRU2Cache(max);
    }

    public void put(Object key, Object value) {
        this.store.put(key, value);
    }

    public Object get(Object key) {
        return this.store.get(key);
    }
}
```

#### lru2Cache 对象

其他低版本是用的`LruCache`，这里是用的`lru2Cache`，区别就是多了一个`PreCache`变量，有点像一级缓存，我这里有点没明白为什么要弄这个`PreCache`???

它继承`LinkedHashMap` ，因为`LinkedHashMap`有个 accessOrder 属性，排序的意思，这里主要作用就是加锁，保证线程安全，除了重写`removeEldestEntry`方法，没做其他事情，因为`LinkedHashMap`和 `hashMap`源码差不多，这里不多介绍，去看`HashMap`源码

```java
public class LRU2Cache<K, V> extends LinkedHashMap<K, V> {
    private static final long serialVersionUID = -5167631809472116969L;
    private static final float DEFAULT_LOAD_FACTOR = 0.75F;
    private static final int DEFAULT_MAX_CAPACITY = 1000;
    private final Lock lock;
    private volatile int maxCapacity;
    //和低版本的区别地方，多的preCache缓存，有点不明白什么作用，没有深入
    private final PreCache<K, Boolean> preCache;

    public LRU2Cache() {
        this(1000);
    }

    public LRU2Cache(int maxCapacity) {
        super(16, 0.75F, true);
        this.lock = new ReentrantLock();
        this.maxCapacity = maxCapacity;
        this.preCache = new PreCache(maxCapacity);
    }

    // 重写 removeEldestEntry 方法，方便移除最老的元素
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return this.size() > this.maxCapacity;
    }

    public boolean containsKey(Object key) {
        this.lock.lock();

        boolean var2;
        try {
            var2 = super.containsKey(key);
        } finally {
            this.lock.unlock();
        }

        return var2;
    }

    public V get(Object key) {
        this.lock.lock();

        Object var2;
        try {
            var2 = super.get(key);
        } finally {
            this.lock.unlock();
        }

        return var2;
    }

    public V put(K key, V value) {
        this.lock.lock();

        Object var3;
        try {
            if (this.preCache.containsKey(key)) {
                this.preCache.remove(key);
                var3 = super.put(key, value);
                return var3;
            }

            this.preCache.put(key, true);
            var3 = value;
        } finally {
            this.lock.unlock();
        }

        return var3;
    }

    public V computeIfAbsent(K key, Function<? super K, ? extends V> fn) {
        V value = this.get(key);
        if (value == null) {
            value = fn.apply(key);
            this.put(key, value);
        }

        return value;
    }

    public V remove(Object key) {
        this.lock.lock();

        Object var2;
        try {
            this.preCache.remove(key);
            var2 = super.remove(key);
        } finally {
            this.lock.unlock();
        }

        return var2;
    }

    public int size() {
        this.lock.lock();

        int var1;
        try {
            var1 = super.size();
        } finally {
            this.lock.unlock();
        }

        return var1;
    }

    public void clear() {
        this.lock.lock();

        try {
            this.preCache.clear();
            super.clear();
        } finally {
            this.lock.unlock();
        }

    }

    public int getMaxCapacity() {
        return this.maxCapacity;
    }

    public void setMaxCapacity(int maxCapacity) {
        this.preCache.setMaxCapacity(maxCapacity);
        this.maxCapacity = maxCapacity;
    }

    static class PreCache<K, V> extends LinkedHashMap<K, V> {
        private volatile int maxCapacity;

        public PreCache() {
            this(1000);
        }

        public PreCache(int maxCapacity) {
            super(16, 0.75F, true);
            this.maxCapacity = maxCapacity;
        }

        protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
            return this.size() > this.maxCapacity;
        }

        public void setMaxCapacity(int maxCapacity) {
            this.maxCapacity = maxCapacity;
        }
    }
}
```

### `lfu`实例

最近最少

#### `lfuCacheFactory` 具体工厂实例

```java
public class LfuCacheFactory extends AbstractCacheFactory {
    public LfuCacheFactory() {
    }

    protected Cache createCache(URL url) {
        return new LfuCache(url);
    }
}
```

#### `lfuCache `实例

```java
public class LfuCache implements Cache {
    private final LFUCache store;

    public LfuCache(URL url) {
        int max = url.getParameter("cache.size", 1000);
        float factor = url.getParameter("cache.evictionFactor", 0.75F);
        this.store = new LFUCache(max, factor);
    }

    public void put(Object key, Object value) {
        this.store.put(key, value);
    }

    public Object get(Object key) {
        return this.store.get(key);
    }
}
```

#### `lfuCache`同名具体实现

```java
public class LFUCache<K, V> {
    private final Map<K, CacheNode<K, V>> map;
    private final CacheDeque<K, V>[] freqTable;
    private final int capacity;
    private final int evictionCount;
    private int curSize;
    private final ReentrantLock lock;
    private static final int DEFAULT_INITIAL_CAPACITY = 1000;
    private static final float DEFAULT_EVICTION_FACTOR = 0.75F;

    public LFUCache() {
        this(1000, 0.75F);
    }

    public LFUCache(final int maxCapacity, final float evictionFactor) {
        //构造函数初始化
        this.curSize = 0;
        this.lock = new ReentrantLock();
        if (maxCapacity <= 0) {
            throw new IllegalArgumentException("Illegal initial capacity: " + maxCapacity);
        } else {
            boolean factorInRange = evictionFactor <= 1.0F && evictionFactor > 0.0F;
            if (factorInRange && !Float.isNaN(evictionFactor)) {
                //最大容量
                this.capacity = maxCapacity;
                //类似hashmap里面的阈值
                this.evictionCount = (int)((float)this.capacity * evictionFactor);
                this.map = new HashMap();
                this.freqTable = new CacheDeque[this.capacity + 1];

                int i;
                //数组元素初始化
                for(i = 0; i <= this.capacity; ++i) {
                    this.freqTable[i] = new CacheDeque();
                }

                //数组元素nextqueue元素赋值
                for(i = 0; i < this.capacity; ++i) {
                    this.freqTable[i].nextDeque = this.freqTable[i + 1];
                }

                this.freqTable[this.capacity].nextDeque = this.freqTable[this.capacity];
            } else {
                throw new IllegalArgumentException("Illegal eviction factor value:" + evictionFactor);
            }
        }
    }

    public int getCapacity() {
        return this.capacity;
    }

    public V put(final K key, final V value) {
        this.lock.lock();

        CacheNode node;
        try {
            //hashmap获取
            node = (CacheNode)this.map.get(key);
            //增加都是往下标为的索引增加
            if (node != null) {
                //删除node节点
                LFUCache.CacheNode.withdrawNode(node);
                node.value = value;
                //增加
                this.freqTable[0].addLastNode(node);
                this.map.put(key, node);
            } else {
                ++this.curSize;
                //容量大于最大容量，删除最近最少使用的元素
                if (this.curSize > this.capacity) {
                    this.proceedEviction();
                }

                //增加
                node = this.freqTable[0].addLast(key, value);
                this.map.put(key, node);
            }
        } finally {
            this.lock.unlock();
        }

        return node.value;
    }

    public V remove(final K key) {
        CacheNode<K, V> node = null;
        this.lock.lock();

        try {
            //移除元素
            if (this.map.containsKey(key)) {
                node = (CacheNode)this.map.remove(key);
                if (node != null) {
                    LFUCache.CacheNode.withdrawNode(node);
                }

                --this.curSize;
            }
        } finally {
            this.lock.unlock();
        }

        return node != null ? node.value : null;
    }

    public V get(final K key) {
        CacheNode<K, V> node = null;
        this.lock.lock();

        try {
            //获取元素，获取之后将元素移至下一索引的队列中
            if (this.map.containsKey(key)) {
                node = (CacheNode)this.map.get(key);
                LFUCache.CacheNode.withdrawNode(node);
                node.owner.nextDeque.addLastNode(node);
            }
        } finally {
            this.lock.unlock();
        }

        return node != null ? node.value : null;
    }

    private int proceedEviction() {
        int targetSize = this.capacity - this.evictionCount;
        int evictedElements = 0;

        //按下标删除节点，直到元素满足阈值，这里下标越靠前，代表使用次数越少
        for(int i = 0; i <= this.capacity; ++i) {
            while(!this.freqTable[i].isEmpty()) {
                CacheNode<K, V> node = this.freqTable[i].pollFirst();
                this.remove(node.key);
                if (targetSize >= this.curSize) {
                    return evictedElements;
                }

                ++evictedElements;
            }
        }

        return evictedElements;
    }

    public int getSize() {
        return this.curSize;
    }

    static class CacheDeque<K, V> {
        CacheNode<K, V> last = new CacheNode();
        CacheNode<K, V> first = new CacheNode();
        //保存node节点
        CacheDeque<K, V> nextDeque;

        CacheDeque() {
            this.last.next = this.first;
            this.first.prev = this.last;
        }

        CacheNode<K, V> addLast(final K key, final V value) {
            CacheNode<K, V> node = new CacheNode(key, value);
            node.owner = this;
            node.next = this.last.next;
            node.prev = this.last;
            node.next.prev = node;
            this.last.next = node;
            return node;
        }

        CacheNode<K, V> addLastNode(final CacheNode<K, V> node) {
            node.owner = this;
            node.next = this.last.next;
            node.prev = this.last;
            node.next.prev = node;
            this.last.next = node;
            return node;
        }

        CacheNode<K, V> pollFirst() {
            CacheNode<K, V> node = null;
            if (this.first.prev != this.last) {
                node = this.first.prev;
                this.first.prev = node.prev;
                this.first.prev.next = this.first;
                node.prev = null;
                node.next = null;
            }

            return node;
        }

        boolean isEmpty() {
            return this.last.next == this.first;
        }
    }

    static class CacheNode<K, V> {
        CacheNode<K, V> prev;
        CacheNode<K, V> next;
        K key;
        V value;
        CacheDeque<K, V> owner;

        CacheNode() {
        }

        CacheNode(final K key, final V value) {
            this.key = key;
            this.value = value;
        }

        static <K, V> CacheNode<K, V> withdrawNode(final CacheNode<K, V> node) {
            //删除当前节点
            if (node != null && node.prev != null) {
                node.prev.next = node.next;
                if (node.next != null) {
                    node.next.prev = node.prev;
                }
            }

            return node;
        }
    }
}
```

### jCache 实例

#### `jCacheFactory` 具体实例工厂

```java
public class JCacheFactory extends AbstractCacheFactory {
    public JCacheFactory() {
    }

    protected Cache createCache(URL url) {
        return new JCache(url);
    }
}
```

#### jCache 具体实现

由于不知道什么原因，看不到CachingProvider以及cacheManager源码，这里先暂时没有笔记~~~

```java
public class JCache implements org.apache.dubbo.cache.Cache {

    private final Cache<Object, Object> store;

    public JCache(URL url) {
        String method = url.getParameter(METHOD_KEY, "");
        String key = url.getAddress() + "." + url.getServiceKey() + "." + method;
        // jcache parameter is the full-qualified class name of SPI implementation
        String type = url.getParameter("jcache");

        CachingProvider provider =
                StringUtils.isEmpty(type) ? Caching.getCachingProvider() : Caching.getCachingProvider(type);
        CacheManager cacheManager = provider.getCacheManager();
        Cache<Object, Object> cache = cacheManager.getCache(key);
        if (cache == null) {
            try {
                // configure the cache
                MutableConfiguration config = new MutableConfiguration<>()
                        .setTypes(Object.class, Object.class)
                        .setExpiryPolicyFactory(CreatedExpiryPolicy.factoryOf(new Duration(
                                TimeUnit.MILLISECONDS,
                                url.getMethodParameter(method, "cache.write.expire", 60 * 1000))))
                        .setStoreByValue(false)
                        .setManagementEnabled(true)
                        .setStatisticsEnabled(true);
                cache = cacheManager.createCache(key, config);
            } catch (CacheException e) {
                // concurrent cache initialization
                cache = cacheManager.getCache(key);
            }
        }

        this.store = cache;
    }

    @Override
    public void put(Object key, Object value) {
        store.put(key, value);
    }

    @Override
    public Object get(Object key) {
        return store.get(key);
    }
}
```

### `expiring `实例

#### `ExpiringCacheFactory`具体工厂实现

```java
public class ExpiringCacheFactory extends AbstractCacheFactory {

    /**
     * Takes url as an method argument and return new instance of cache store implemented by JCache.
     * @param url url of the method
     * @return ExpiringCache instance of cache
     */
    @Override
    protected Cache createCache(URL url) {
        return new ExpiringCache(url);
    }
}
```

#### `expiring ` 实例

```java
public class ExpiringCache implements Cache {
    private final Map<Object, Object> store;

    public ExpiringCache(URL url) {
        //获取参数
        // cache time (second)
        final int secondsToLive = url.getParameter("cache.seconds", 180);
        // Cache check interval (second)
        final int intervalSeconds = url.getParameter("cache.interval", 4);
        //创建map
        ExpiringMap<Object, Object> expiringMap = new ExpiringMap<>(secondsToLive, intervalSeconds);
        //启动守护线程
        expiringMap.getExpireThread().startExpiryIfNotStarted();
        this.store = expiringMap;
    }

    /**
     * API to store value against a key in the calling thread scope.
     * @param key  Unique identifier for the object being store.
     * @param value Value getting store
     */
    @Override
    public void put(Object key, Object value) {
        store.put(key, value);
    }

    /**
     * API to return stored value using a key against the calling thread specific store.
     * @param key Unique identifier for cache lookup
     * @return Return stored object against key
     */
    @Override
    public Object get(Object key) {
        return store.get(key);
    }
}
```

#### `ExpiringMap`具体实现

它是Map的子类，包了一层

```java
public class ExpiringMap<K, V> implements Map<K, V> {

    /**
     * default time to live (second)
     */
    //默认存活时间
    private static final int DEFAULT_TIME_TO_LIVE = 180;

    /**
     * default expire check interval (second)
     */
    //默认过期时间
    private static final int DEFAULT_EXPIRATION_INTERVAL = 1;

    //过期数量
    private static final AtomicInteger expireCount = new AtomicInteger(1);

    //存储空间
    private final ConcurrentHashMap<K, ExpiryObject> delegateMap;

    //守护线程
    private final ExpireThread expireThread;

    public ExpiringMap() {
        this(DEFAULT_TIME_TO_LIVE, DEFAULT_EXPIRATION_INTERVAL);
    }

    /**
     * Constructor
     *
     * @param timeToLive time to live (second)
     */
    public ExpiringMap(int timeToLive) {
        this(timeToLive, DEFAULT_EXPIRATION_INTERVAL);
    }

    public ExpiringMap(int timeToLive, int expirationInterval) {
        this(new ConcurrentHashMap<>(), timeToLive, expirationInterval);
    }

    private ExpiringMap(ConcurrentHashMap<K, ExpiryObject> delegateMap, int timeToLive, int expirationInterval) {
        //构造函数初始化map，守护线程，存活时间，过期时间
        this.delegateMap = delegateMap;
        this.expireThread = new ExpireThread();
        expireThread.setTimeToLive(timeToLive);
        expireThread.setExpirationInterval(expirationInterval);
    }

    @Override
    public V put(K key, V value) {
        //设置key，value，以及最后访问时间，第一次设置值为当前时间
        ExpiryObject answer = delegateMap.put(key, new ExpiryObject(key, value, System.currentTimeMillis()));
        if (answer == null) {
            return null;
        }
        return answer.getValue();
    }

    @Override
    public V get(Object key) {
        //获取值
        ExpiryObject object = delegateMap.get(key);
        if (object != null) {
            //计算存活时间，是否超时，超时就删除，这里是兜底操作，为了防止异步线程没有及时情理 
            long timeIdle = System.currentTimeMillis() - object.getLastAccessTime();
            int timeToLive = expireThread.getTimeToLive();
            if (timeToLive > 0 && timeIdle >= timeToLive * 1000L) {
                delegateMap.remove(object.getKey());
                return null;
            }
            object.setLastAccessTime(System.currentTimeMillis());
            return object.getValue();
        }
        return null;
    }

    @Override
    public V remove(Object key) {
        ExpiryObject answer = delegateMap.remove(key);
        if (answer == null) {
            return null;
        }
        return answer.getValue();
    }

    @Override
    public boolean containsKey(Object key) {
        return delegateMap.containsKey(key);
    }

    @Override
    public boolean containsValue(Object value) {
        return delegateMap.containsValue(value);
    }

    @Override
    public int size() {
        return delegateMap.size();
    }

    @Override
    public boolean isEmpty() {
        return delegateMap.isEmpty();
    }

    @Override
    public void clear() {
        delegateMap.clear();
        expireThread.stopExpiring();
    }

    @Override
    public int hashCode() {
        return delegateMap.hashCode();
    }

    @Override
    public Set<K> keySet() {
        return delegateMap.keySet();
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }
        return delegateMap.equals(obj);
    }

    @Override
    public void putAll(Map<? extends K, ? extends V> inMap) {
        for (Entry<? extends K, ? extends V> e : inMap.entrySet()) {
            this.put(e.getKey(), e.getValue());
        }
    }

    @Override
    public Collection<V> values() {
        List<V> list = new ArrayList<>();
        Set<Entry<K, ExpiryObject>> delegatedSet = delegateMap.entrySet();
        for (Entry<K, ExpiryObject> entry : delegatedSet) {
            ExpiryObject value = entry.getValue();
            list.add(value.getValue());
        }
        return list;
    }

    @Override
    public Set<Entry<K, V>> entrySet() {
        throw new UnsupportedOperationException();
    }

    public ExpireThread getExpireThread() {
        return expireThread;
    }

    public int getExpirationInterval() {
        return expireThread.getExpirationInterval();
    }

    public void setExpirationInterval(int expirationInterval) {
        expireThread.setExpirationInterval(expirationInterval);
    }

    public int getTimeToLive() {
        return expireThread.getTimeToLive();
    }

    public void setTimeToLive(int timeToLive) {
        expireThread.setTimeToLive(timeToLive);
    }

    @Override
    public String toString() {
        return "ExpiringMap{" + "delegateMap="
                + delegateMap.toString() + ", expireThread="
                + expireThread.toString() + '}';
    }

    /**
     * can be expired object
     */
    private class ExpiryObject {
        private K key;
        private V value;
        private AtomicLong lastAccessTime;

        //具体值对象，存储键值以及上次访问时间
        ExpiryObject(K key, V value, long lastAccessTime) {
            if (value == null) {
                throw new IllegalArgumentException("An expiring object cannot be null.");
            }
            this.key = key;
            this.value = value;
            this.lastAccessTime = new AtomicLong(lastAccessTime);
        }

        public long getLastAccessTime() {
            return lastAccessTime.get();
        }

        public void setLastAccessTime(long lastAccessTime) {
            this.lastAccessTime.set(lastAccessTime);
        }

        public K getKey() {
            return key;
        }

        public V getValue() {
            return value;
        }

        @Override
        public boolean equals(Object obj) {
            if (this == obj) {
                return true;
            }
            return value.equals(obj);
        }

        @Override
        public int hashCode() {
            return value.hashCode();
        }

        @Override
        public String toString() {
            return "ExpiryObject{" + "key=" + key + ", value=" + value + ", lastAccessTime=" + lastAccessTime + '}';
        }
    }

    /**
     * Background thread, periodically checking if the data is out of date
     */
    public class ExpireThread implements Runnable {
        private long timeToLiveMillis;
        private long expirationIntervalMillis;
        private volatile boolean running = false;
        private final Thread expirerThread;

        @Override
        public String toString() {
            return "ExpireThread{" + ", timeToLiveMillis="
                    + timeToLiveMillis + ", expirationIntervalMillis="
                    + expirationIntervalMillis + ", running="
                    + running + ", expirerThread="
                    + expirerThread + '}';
        }

        public ExpireThread() {
            expirerThread = new Thread(this, "ExpiryMapExpire-" + expireCount.getAndIncrement());
            expirerThread.setDaemon(true);
        }

        @Override
        public void run() {
            while (running) {
                processExpires();
                try {
                    //每多少秒执行一次
                    Thread.sleep(expirationIntervalMillis);
                } catch (InterruptedException e) {
                    running = false;
                }
            }
        }

        private void processExpires() {
            long timeNow = System.currentTimeMillis();
            if (timeToLiveMillis <= 0) {
                return;
            }
            //清除超时过期的值对象
            for (ExpiryObject o : delegateMap.values()) {
                long timeIdle = timeNow - o.getLastAccessTime();
                if (timeIdle >= timeToLiveMillis) {
                    delegateMap.remove(o.getKey());
                }
            }
        }

        /**
         * start expiring Thread
         */
        public void startExpiring() {
            //异步线程开始
            if (!running) {
                running = true;
                expirerThread.start();
            }
        }

        /**
         * start thread
         */
        public void startExpiryIfNotStarted() {
            if (running && timeToLiveMillis <= 0) {
                return;
            }
            startExpiring();
        }

        /**
         * stop thread
         */
        public void stopExpiring() {
            //终止异步线程
            if (running) {
                running = false;
                expirerThread.interrupt();
            }
        }

        /**
         * get thread state
         *
         * @return thread state
         */
        public boolean isRunning() {
            return running;
        }

        /**
         * get time to live
         *
         * @return time to live
         */
        public int getTimeToLive() {
            return (int) timeToLiveMillis / 1000;
        }

        /**
         * update time to live
         *
         * @param timeToLive time to live
         */
        public void setTimeToLive(long timeToLive) {
            this.timeToLiveMillis = timeToLive * 1000;
        }

        /**
         * get expiration interval
         *
         * @return expiration interval (second)
         */
        public int getExpirationInterval() {
            return (int) expirationIntervalMillis / 1000;
        }

        /**
         * set expiration interval
         *
         * @param expirationInterval expiration interval (second)
         */
        public void setExpirationInterval(long expirationInterval) {
            this.expirationIntervalMillis = expirationInterval * 1000;
        }
    }
}
```

