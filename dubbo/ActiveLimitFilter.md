## ActiveLimitFilter 

简介：最大并行数量控制器

```java
@Activate(group = CONSUMER, value = ACTIVES_KEY)
public class ActiveLimitFilter implements Filter, Filter.Listener {

    private static final String ACTIVE_LIMIT_FILTER_START_TIME = "active_limit_filter_start_time";

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        //获取具体并发控制器
        URL url = invoker.getUrl();
        String methodName = RpcUtils.getMethodName(invocation);
        int max = invoker.getUrl().getMethodParameter(methodName, ACTIVES_KEY, 0);
        final RpcStatus rpcStatus = RpcStatus.getStatus(invoker.getUrl(), RpcUtils.getMethodName(invocation));
        //判断是否超过限制，以及在未超过限制的情况下进行属性设置
        if (!RpcStatus.beginCount(url, methodName, max)) {
            //这里是超过限制，最请求进行最后的重试操作
            long timeout = invoker.getUrl().getMethodParameter(RpcUtils.getMethodName(invocation), TIMEOUT_KEY, 0);
            long start = System.currentTimeMillis();
            long remain = timeout;
            //加锁，这里锁的是并发控制器里面的rpcStatus，对同一个rpcStatus所对应的请求进行限制
            synchronized (rpcStatus) {
                while (!RpcStatus.beginCount(url, methodName, max)) {
                    try {
                        //使用对象的wait操作，等待remain秒
                        rpcStatus.wait(remain);
                    } catch (InterruptedException e) {
                        // ignore
                    }
                    long elapsed = System.currentTimeMillis() - start;
                    remain = timeout - elapsed;
                    //如果超时就结束
                    if (remain <= 0) {
                        throw new RpcException(
                                RpcException.LIMIT_EXCEEDED_EXCEPTION,
                                "Waiting concurrent invoke timeout in client-side for service:  "
                                        + invoker.getInterface().getName()
                                        + ", method: " + RpcUtils.getMethodName(invocation) + ", elapsed: "
                                        + elapsed + ", timeout: " + timeout + ". concurrent invokes: "
                                        + rpcStatus.getActive()
                                        + ". max concurrent invoke limit: " + max);
                    }
                }
            }
        }

        //未操作并发控制器的限制
        invocation.put(ACTIVE_LIMIT_FILTER_START_TIME, System.currentTimeMillis());
        //执行方法
        return invoker.invoke(invocation);
    }

}
```



#### RpcStatus并发控制器

```java
public class RpcStatus {

    //存储应用级别并发
    private static final ConcurrentMap<String, RpcStatus> SERVICE_STATISTICS = new ConcurrentHashMap<>();

    //存储方法级别的并发
    private static final ConcurrentMap<String, ConcurrentMap<String, RpcStatus>> METHOD_STATISTICS =
            new ConcurrentHashMap<>();

    private final ConcurrentMap<String, Object> values = new ConcurrentHashMap<>();

    //使用juc下的automic类，确保线程安全
    //调用中的次数
    private final AtomicInteger active = new AtomicInteger();
    //总请求数
    private final AtomicLong total = new AtomicLong();
    //失败的个数
    private final AtomicInteger failed = new AtomicInteger();
    //总调用时间
    private final AtomicLong totalElapsed = new AtomicLong();
    //总失败的时间
    private final AtomicLong failedElapsed = new AtomicLong();
    //最大调用时长
    private final AtomicLong maxElapsed = new AtomicLong();
    //失败的最大时长
    private final AtomicLong failedMaxElapsed = new AtomicLong();
    //成功的最大时长
    private final AtomicLong succeededMaxElapsed = new AtomicLong();

    private RpcStatus() {}

    /**
     * @param url
     * @return status
     */
    public static RpcStatus getStatus(URL url) {
        //获取RpcStatus
        String uri = url.toIdentityString();
        return ConcurrentHashMapUtils.computeIfAbsent(SERVICE_STATISTICS, uri, key -> new RpcStatus());
    }

    /**
     * @param url
     */
    public static void removeStatus(URL url) {
        //移除RpcStatus
        String uri = url.toIdentityString();
        SERVICE_STATISTICS.remove(uri);
    }

    /**
     * @param url
     * @param methodName
     * @return status
     */
    public static RpcStatus getStatus(URL url, String methodName) {
        //方法级别的并发控制器
        String uri = url.toIdentityString();
        ConcurrentMap<String, RpcStatus> map =
                ConcurrentHashMapUtils.computeIfAbsent(METHOD_STATISTICS, uri, k -> new ConcurrentHashMap<>());
        return ConcurrentHashMapUtils.computeIfAbsent(map, methodName, k -> new RpcStatus());
    }

    /**
     * @param url
     */
    public static void removeStatus(URL url, String methodName) {
        String uri = url.toIdentityString();
        ConcurrentMap<String, RpcStatus> map = METHOD_STATISTICS.get(uri);
        if (map != null) {
            map.remove(methodName);
        }
    }

    public static void beginCount(URL url, String methodName) {
        beginCount(url, methodName, Integer.MAX_VALUE);
    }

    /**
     * @param url
     */
    public static boolean beginCount(URL url, String methodName, int max) {
        max = (max <= 0) ? Integer.MAX_VALUE : max;
        //应用级别的并发控制器
        RpcStatus appStatus = getStatus(url);
        //方法级别的并发控制器
        RpcStatus methodStatus = getStatus(url, methodName);
        if (methodStatus.active.get() == Integer.MAX_VALUE) {
            //并发控制器达到最大请求，达到极限
            return false;
        }
        //无限循环，直到cas成功或者请求数达到极限
        for (int i; ; ) {
            i = methodStatus.active.get();

            //当前请求数
            if (i == Integer.MAX_VALUE || i + 1 > max) {
                return false;
            }

            //方法并发控制器cas操作
            if (methodStatus.active.compareAndSet(i, i + 1)) {
                break;
            }
        }

        //应用并发控制器自增
        appStatus.active.incrementAndGet();

        return true;
    }

    /**
     * @param url
     * @param elapsed
     * @param succeeded
     */
    public static void endCount(URL url, String methodName, long elapsed, boolean succeeded) {
        //执行结束
        endCount(getStatus(url), elapsed, succeeded);
        endCount(getStatus(url, methodName), elapsed, succeeded);
    }

    private static void endCount(RpcStatus status, long elapsed, boolean succeeded) {
        //请求结束进行自减
        status.active.decrementAndGet();
        //请求总数+1
        status.total.incrementAndGet();
        //总调用时长增加
        status.totalElapsed.addAndGet(elapsed);

        if (status.maxElapsed.get() < elapsed) {
            status.maxElapsed.set(elapsed);
        }

        if (succeeded) {
            //执行成功的情况，设置最大调用时长
            if (status.succeededMaxElapsed.get() < elapsed) {
                status.succeededMaxElapsed.set(elapsed);
            }

        } else {
            //执行失败的情况，失败请求数+1，失败时长自增
            status.failed.incrementAndGet();
            status.failedElapsed.addAndGet(elapsed);
            if (status.failedMaxElapsed.get() < elapsed) {
                //设置最大失败时长
                status.failedMaxElapsed.set(elapsed);
            }
        }
    }

    /**
     * set value.
     *
     * @param key
     * @param value
     */
    public void set(String key, Object value) {
        values.put(key, value);
    }

    /**
     * get value.
     *
     * @param key
     * @return value
     */
    public Object get(String key) {
        return values.get(key);
    }

    /**
     * get active.
     *
     * @return active
     */
    public int getActive() {
        return active.get();
    }

    /**
     * get total.
     *
     * @return total
     */
    public long getTotal() {
        return total.longValue();
    }

    /**
     * get total elapsed.
     *
     * @return total elapsed
     */
    public long getTotalElapsed() {
        return totalElapsed.get();
    }

    /**
     * get average elapsed.
     *
     * @return average elapsed
     */
    public long getAverageElapsed() {
        long total = getTotal();
        if (total == 0) {
            return 0;
        }
        return getTotalElapsed() / total;
    }

    /**
     * get max elapsed.
     *
     * @return max elapsed
     */
    public long getMaxElapsed() {
        return maxElapsed.get();
    }

    /**
     * get failed.
     *
     * @return failed
     */
    public int getFailed() {
        return failed.get();
    }

    /**
     * get failed elapsed.
     *
     * @return failed elapsed
     */
    public long getFailedElapsed() {
        return failedElapsed.get();
    }

    /**
     * get failed average elapsed.
     *
     * @return failed average elapsed
     */
    public long getFailedAverageElapsed() {
        long failed = getFailed();
        if (failed == 0) {
            return 0;
        }
        return getFailedElapsed() / failed;
    }

    /**
     * get failed max elapsed.
     *
     * @return failed max elapsed
     */
    public long getFailedMaxElapsed() {
        return failedMaxElapsed.get();
    }

    /**
     * get succeeded.
     *
     * @return succeeded
     */
    public long getSucceeded() {
        return getTotal() - getFailed();
    }

    /**
     * get succeeded elapsed.
     *
     * @return succeeded elapsed
     */
    public long getSucceededElapsed() {
        return getTotalElapsed() - getFailedElapsed();
    }

    /**
     * get succeeded average elapsed.
     *
     * @return succeeded average elapsed
     */
    public long getSucceededAverageElapsed() {
        long succeeded = getSucceeded();
        if (succeeded == 0) {
            return 0;
        }
        return getSucceededElapsed() / succeeded;
    }

    /**
     * get succeeded max elapsed.
     *
     * @return succeeded max elapsed.
     */
    public long getSucceededMaxElapsed() {
        return succeededMaxElapsed.get();
    }

    /**
     * Calculate average TPS (Transaction per second).
     *
     * @return tps
     */
    public long getAverageTps() {
        if (getTotalElapsed() >= 1000L) {
            return getTotal() / (getTotalElapsed() / 1000L);
        }
        return getTotal();
    }
}
```

