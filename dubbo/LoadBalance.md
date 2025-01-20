## LoadBalance

简介：负载均衡算法

#### `LoadBalance`接口

定义负载均衡核心功能，由抽象类继承该接口，实现策略以及模板

```java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {

    /**
     * select one invoker in list.
     *
     * @param invokers   invokers.
     * @param url        refer url
     * @param invocation invocation.
     * @return selected invoker.
     */
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```

#### `AbstractLoadBalance`抽象类

简介：实现接口，定义子类公共方法，以及抽象模板，常见的设计模式

```java
public abstract class AbstractLoadBalance implements LoadBalance {
    /**
     * Calculate the weight according to the uptime proportion of warmup time
     * the new weight will be within 1(inclusive) to weight(inclusive)
     *
     * @param uptime the uptime in milliseconds
     * @param warmup the warmup time in milliseconds
     * @param weight the weight of an invoker
     * @return weight which takes warmup into account
     */
    static int calculateWarmupWeight(int uptime, int warmup, int weight) {
        // 由于jvm的预热机制，启动时间/需要的总预热时间 * 权重比例
        // 这里的意思就是根据启动时间的多少然后分配相应的流量
        int ww = (int) (uptime / ((float) warmup / weight));
        return ww < 1 ? 1 : (Math.min(ww, weight));
    }

    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        // 直接获取第一个
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
        // 抽象方法，交由子类实现
        return doSelect(invokers, url, invocation);
    }

    protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);

    /**
     * Get the weight of the invoker's invocation which takes warmup time into account
     * if the uptime is within the warmup time, the weight will be reduce proportionally
     *
     * @param invoker    the invoker
     * @param invocation the invocation of this invoker
     * @return weight
     */
    protected int getWeight(Invoker<?> invoker, Invocation invocation) {
        // 获取权重的方法
        int weight;
        URL url = invoker.getUrl();
        if (invoker instanceof ClusterInvoker) {
            url = ((ClusterInvoker<?>) invoker).getRegistryUrl();
        }

        // Multiple registry scenario, load balance among multiple registries.
        // 直接获取权重
        if (REGISTRY_SERVICE_REFERENCE_PATH.equals(url.getServiceInterface())) {
            weight = url.getParameter(WEIGHT_KEY, DEFAULT_WEIGHT);
        } else {
            // 获取权重
            weight = url.getMethodParameter(RpcUtils.getMethodName(invocation), WEIGHT_KEY, DEFAULT_WEIGHT);
            if (weight > 0) {
                // 获取启动时间
                long timestamp = invoker.getUrl().getParameter(TIMESTAMP_KEY, 0L);
                if (timestamp > 0L) {
                    // 计算已经启动时间
                    long uptime = System.currentTimeMillis() - timestamp;
                    // 当前时间小于启动时间，就直接返回 1
                    if (uptime < 0) {
                        return 1;
                    }
                    // 获取需要预热的时间，如果已经启动时间小于需要预热的时间
                    int warmup = invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP);
                    if (uptime > 0 && uptime < warmup) {
                        weight = calculateWarmupWeight((int) uptime, warmup, weight);
                    }
                }
            }
        }
        return Math.max(weight, 0);
    }
}
```



#### RandomLoadBalance具体实现类

```java
public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";

    /**
     * Select one invoker between a list using a random criteria
     *
     * @param invokers   List of possible invokers
     * @param url        URL
     * @param invocation Invocation
     * @param <T>
     * @return The selected invoker
     */
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        int length = invokers.size();

        if (!needWeightLoadBalance(invokers, invocation)) {
            // 不需要按照权重分配，直接随机数
            return invokers.get(ThreadLocalRandom.current().nextInt(length));
        }

        // Every invoker has the same weight?
        // 每个服务的权重是否相同
        boolean sameWeight = true;
        // the maxWeight of every invoker, the minWeight = 0 or the maxWeight of the last invoker
        // 保存每个服务的权重区间
        int[] weights = new int[length];
        // The sum of weights
        int totalWeight = 0;
        for (int i = 0; i < length; i++) {
            // 调用抽象父类的获取权重的方法
            int weight = getWeight(invokers.get(i), invocation);
            // Sum
            // 总权重
            totalWeight += weight;
            // save for later use
            // 设置权重
            weights[i] = totalWeight;
            if (sameWeight && totalWeight != weight * (i + 1)) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on
            // totalWeight.
            // 权重不相同的情况下，先获取到随机数
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return an invoker based on the random value.
            // 服务的个数小于4，直接判断权重在哪个区间
            if (length <= 4) {
                for (int i = 0; i < length; i++) {
                    if (offset < weights[i]) {
                        return invokers.get(i);
                    }
                }
            } else {
                // 服务个数大于等于4，二分查找服务
                int i = Arrays.binarySearch(weights, offset);
                if (i < 0) {
                    //服务不存在
                    i = -i - 1;
                } else {
                    //服务存在的情况
                    while (weights[i + 1] == offset) {
                        i++;
                    }
                    i++;
                }
                return invokers.get(i);
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        // 权重相同的情况下，随机分配
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }

    private <T> boolean needWeightLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {
        Invoker<T> invoker = invokers.get(0);
        URL invokerUrl = invoker.getUrl();
        if (invoker instanceof ClusterInvoker) {
            invokerUrl = ((ClusterInvoker<?>) invoker).getRegistryUrl();
        }

        // Multiple registry scenario, load balance among multiple registries.
        // 判断权重标识是否存在
        if (REGISTRY_SERVICE_REFERENCE_PATH.equals(invokerUrl.getServiceInterface())) {
            String weight = invokerUrl.getParameter(WEIGHT_KEY);
            return StringUtils.isNotEmpty(weight);
        } else {
            String weight = invokerUrl.getMethodParameter(RpcUtils.getMethodName(invocation), WEIGHT_KEY);
            if (StringUtils.isNotEmpty(weight)) {
                return true;
            } else {
                String timeStamp = invoker.getUrl().getParameter(TIMESTAMP_KEY);
                return StringUtils.isNotEmpty(timeStamp);
            }
        }
    }
}
```



#### RoundRobinLoadBalance具体实现类

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "roundrobin";

    private static final int RECYCLE_PERIOD = 60000;

    // 存储服务信息
    private final ConcurrentMap<String, ConcurrentMap<String, WeightedRoundRobin>> methodWeightMap =
            new ConcurrentHashMap<>();

    protected static class WeightedRoundRobin {
        // 权重
        private int weight;
        // 当前访问的次数
        private final AtomicLong current = new AtomicLong(0);
        // 最后访问时间
        private long lastUpdate;

        public int getWeight() {
            return weight;
        }

        public void setWeight(int weight) {
            this.weight = weight;
            current.set(0);
        }

        public long increaseCurrent() {
            return current.addAndGet(weight);
        }

        public void sel(int total) {
            current.addAndGet(-1 * total);
        }

        public long getLastUpdate() {
            return lastUpdate;
        }

        public void setLastUpdate(long lastUpdate) {
            this.lastUpdate = lastUpdate;
        }
    }

    /**
     * get invoker addr list cached for specified invocation
     * <p>
     * <b>for unit test only</b>
     *
     * @param invokers
     * @param invocation
     * @return
     */
    protected <T> Collection<String> getInvokerAddrList(List<Invoker<T>> invokers, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + RpcUtils.getMethodName(invocation);
        Map<String, WeightedRoundRobin> map = methodWeightMap.get(key);
        if (map != null) {
            return map.keySet();
        }
        return null;
    }

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // 获取外部的key
        String key = invokers.get(0).getUrl().getServiceKey() + "." + RpcUtils.getMethodName(invocation);
        ConcurrentMap<String, WeightedRoundRobin> map =
                ConcurrentHashMapUtils.computeIfAbsent(methodWeightMap, key, k -> new ConcurrentHashMap<>());
        // 总权重
        int totalWeight = 0;
        // 最大权重
        long maxCurrent = Long.MIN_VALUE;
        long now = System.currentTimeMillis();
        // 选中的服务
        Invoker<T> selectedInvoker = null;
        // 选中服务所对应的权重信息
        WeightedRoundRobin selectedWRR = null;
        for (Invoker<T> invoker : invokers) {
            // 获取内部key
            String identifyString = invoker.getUrl().toIdentityString();
            // 抽象父类方法获取权重
            int weight = getWeight(invoker, invocation);
            // 权重信息存储
            WeightedRoundRobin weightedRoundRobin = ConcurrentHashMapUtils.computeIfAbsent(map, identifyString, k -> {
                WeightedRoundRobin wrr = new WeightedRoundRobin();
                wrr.setWeight(weight);
                return wrr;
            });

            if (weight != weightedRoundRobin.getWeight()) {
                // weight changed
                // 权重信息改变的情况下，重新赋值
                weightedRoundRobin.setWeight(weight);
            }
            // 将当前权重信息自增
            long cur = weightedRoundRobin.increaseCurrent();
            // 设置最后访问时间
            weightedRoundRobin.setLastUpdate(now);
            // 选择当前权重信息最大的服务
            if (cur > maxCurrent) {
                maxCurrent = cur;
                selectedInvoker = invoker;
                selectedWRR = weightedRoundRobin;
            }
            totalWeight += weight;
        }
        // 服务的个数不等于存储的权重信息个数，删除 最后访问时间 < 当前时间 - 区间
        if (invokers.size() != map.size()) {
            map.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
        }
        if (selectedInvoker != null) {
            // 重置当前权重信息的权重属性，为了使其他服务能够被调用
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }
        // should not happen here
        return invokers.get(0);
    }
}
```



#### LeastActiveLoadBalance 具体实现类

简介：最少活跃数

```java
public class LeastActiveLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "leastactive";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        // 服务个数
        int length = invokers.size();
        // The least active value of all invokers
        // 当前最小的活跃数
        int leastActive = -1;
        // The number of invokers having the same least active value (leastActive)
        // 当前相同的活跃数的服务个数
        int leastCount = 0;
        // The index of invokers having the same least active value (leastActive)
        // 保存最小服务所对应的下标
        int[] leastIndexes = new int[length];
        // the weight of every invokers
        // 保存权重
        int[] weights = new int[length];
        // The sum of the warmup weights of all the least active invokers
        // 权重总和
        int totalWeight = 0;
        // The weight of the first least active invoker
        // 当前最低的权重
        int firstWeight = 0;
        // Every least active invoker has the same weight value?
        boolean sameWeight = true;

        // Filter out all the least active invokers
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            // Get the active number of the invoker
            // RpcStatus 这个是获取活跃数，具体看activeLimitFilter 源码笔记
            int active = RpcStatus.getStatus(invoker.getUrl(), RpcUtils.getMethodName(invocation))
                    .getActive();
            // Get the weight of the invoker's configuration. The default value is 100.
            // 获取权重
            int afterWarmup = getWeight(invoker, invocation);
            // save for later use
            // 保存权重
            weights[i] = afterWarmup;
            // If it is the first invoker or the active number of the invoker is less than the current least active
            // number
            if (leastActive == -1 || active < leastActive) {
                // 保存最低活跃数
                // Reset the active number of the current invoker to the least active number
                leastActive = active;
                // Reset the number of least active invokers
                // 相同最低活跃数的服务个数
                leastCount = 1;
                // Put the first least active invoker first in leastIndexes
                // 保存服务下标
                leastIndexes[0] = i;
                // Reset totalWeight
                totalWeight = afterWarmup;
                // Record the weight the first least active invoker
                firstWeight = afterWarmup;
                // Each invoke has the same weight (only one invoker here)
                sameWeight = true;
                // If current invoker's active value equals with leaseActive, then accumulating.
            } else if (active == leastActive) {
                // 多个相同活跃数的服务
                // Record the index of the least active invoker in leastIndexes order
                leastIndexes[leastCount++] = i;
                // Accumulate the total weight of the least active invoker
                totalWeight += afterWarmup;
                // If every invoker has the same weight?
                if (sameWeight && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // Choose an invoker from all the least active invokers
        // 只有一个服务
        if (leastCount == 1) {
            // If we got exactly one invoker having the least active value, return this invoker directly.
            return invokers.get(leastIndexes[0]);
        }
        // 多个服务，并且权重不一样，就区间取值
        if (!sameWeight && totalWeight > 0) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on
            // totalWeight.
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        // 多个服务，权重一致的情况下，随机取值
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
    }
}
```

