## `TpsLimitFilter`

简介：限流过滤器

```java
@Activate(group = CommonConstants.PROVIDER, value = TPS_LIMIT_RATE_KEY)
public class TpsLimitFilter implements Filter {

    //默认限流器
    private final TPSLimiter tpsLimiter = new DefaultTPSLimiter();

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {

        //调用tpsLimiter的isAllowable方法来判断是否限流
        if (!tpsLimiter.isAllowable(invoker.getUrl(), invocation)) {
            return AsyncRpcResult.newDefaultAsyncResult(
                    new RpcException(
                            "Failed to invoke service " + invoker.getInterface().getName() + "."
                                    + RpcUtils.getMethodName(invocation) + " because exceed max service tps."),
                    invocation);
        }

        return invoker.invoke(invocation);
    }
}
```

### 默认限流器

```java
public class DefaultTPSLimiter implements TPSLimiter {

    //map 存储对应限流器
    private final ConcurrentMap<String, StatItem> stats = new ConcurrentHashMap<>();

    @Override
    public boolean isAllowable(URL url, Invocation invocation) {
        //频率
        int rate = url.getMethodParameter(RpcUtils.getMethodName(invocation), TPS_LIMIT_RATE_KEY, -1);
        //时间间隔
        long interval = url.getMethodParameter(
                RpcUtils.getMethodName(invocation), TPS_LIMIT_INTERVAL_KEY, DEFAULT_TPS_LIMIT_INTERVAL);
        //缓存key
        String serviceKey = url.getServiceKey();
        if (rate > 0) {
            StatItem statItem = stats.get(serviceKey);
            //不存在增加
            if (statItem == null) {
                stats.putIfAbsent(serviceKey, new StatItem(serviceKey, rate, interval));
                statItem = stats.get(serviceKey);
            } else {
                // rate or interval has changed, rebuild
                //存在的情况，如果配置变更，这里需要更新限流器
                if (statItem.getRate() != rate || statItem.getInterval() != interval) {
                    stats.put(serviceKey, new StatItem(serviceKey, rate, interval));
                    statItem = stats.get(serviceKey);
                }
            }
            return statItem.isAllowable();
        } else {
            //不需要限流，移除限流器
            StatItem statItem = stats.get(serviceKey);
            if (statItem != null) {
                stats.remove(serviceKey);
            }
        }

        return true;
    }
}
```

### 限流器具体实现

`AtomicInteger`内容去看 `AtomicInteger`源码笔记~~~

```java
class StatItem {

    //限流器名称
    private final String name;

    //上次设置的时间
    private final AtomicLong lastResetTime;

    //时间窗口
    private final long interval;

    //token，也就是频率
    private final AtomicInteger token;

    //频率
    private final int rate;

    StatItem(String name, int rate, long interval) {
        this.name = name;
        this.rate = rate;
        this.interval = interval;
        this.lastResetTime = new AtomicLong(System.currentTimeMillis());
        this.token = new AtomicInteger(rate);
    }

    public boolean isAllowable() {
        //当前时间
        long now = System.currentTimeMillis();
        //如果已经超时了，就重置限流器里面的属性
        if (now > lastResetTime.get() + interval) {
            token.set(rate);
            lastResetTime.set(now);
        }

        //调用AtomicInteger的自减方法
        return token.decrementAndGet() >= 0;
    }

    public long getInterval() {
        return interval;
    }

    public int getRate() {
        return rate;
    }

    long getLastResetTime() {
        return lastResetTime.get();
    }

    int getToken() {
        return token.get();
    }

    @Override
    public String toString() {
        return "StatItem " + "[name=" + name + ", " + "rate = " + rate + ", " + "interval = " + interval + ']';
    }
}
```

