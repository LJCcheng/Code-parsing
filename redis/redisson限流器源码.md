## Redisson 限流器源码

主要看 RedissonRateLimiter 类，它实现了 `RRateLimiter`接口和`RRateLimiterAsync`。`RRateLimiter`接口是同步方法，`RRateLimiterAsync`接口是异步方法，方法都差不多，区别就是一个是同步另一个是异步。

#### tryAcquire()方法

```java
public boolean tryAcquire(long permits) {
    	//这里的get是获取tryAcquireAsync的返回，tryAcquireAsync的返回是RFuture，对Future进行了封装
        return get(tryAcquireAsync(RedisCommands.EVAL_NULL_BOOLEAN, permits));
    }

private <T> RFuture<T> tryAcquireAsync(RedisCommand<T> command, Long value) {
        byte[] random = new byte[8];
        ThreadLocalRandom.current().nextBytes(random);

    	//执行lua脚本
        return commandExecutor.evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
               //获取限流配置，速度，时间窗口，限流类型
                "local rate = redis.call('hget', KEYS[1], 'rate');"
              + "local interval = redis.call('hget', KEYS[1], 'interval');"
              + "local type = redis.call('hget', KEYS[1], 'type');"
              + "assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')"
                                              
              //判断是单机限流还是分布式限流，设置不同key
              + "local valueName = KEYS[2];"
              + "local permitsName = KEYS[4];"
              + "if type == '1' then "
                  + "valueName = KEYS[3];"
                  + "permitsName = KEYS[5];"
              + "end;"

              // rate速度必须必请求的ARGV[1]大                           
              + "assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); "

              //获取当前的String结构的值                               
              + "local currentValue = redis.call('get', valueName); "
              + "if currentValue ~= false then "
                       //获取得到值
                       //统计过期令牌                       
                     + "local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); "
                     + "local released = 0; "
                     //循环计算过期的令牌之和                         
                     + "for i, v in ipairs(expiredValues) do "
                          + "local random, permits = struct.unpack('Lc0I', v);"
                          + "released = released + permits;"
                     + "end; "

                     + "if released > 0 then "
                             //删除过期数据                   
                          + "redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); "
                          + "if tonumber(currentValue) + released > tonumber(rate) then "
                                 //如果当前令牌加上之前释放的令牌大于rate，就用rate减去当前已经使用的令牌数
                               + "currentValue = tonumber(rate) - redis.call('zcard', permitsName); "
                          + "else "
                                 //否则就用当前剩余令牌加上已经释放的令牌数              
                               + "currentValue = tonumber(currentValue) + released; "
                          + "end; "
                            //重新设值                  
                          + "redis.call('set', valueName, currentValue);"
                     + "end;"
                       
                     + "if tonumber(currentValue) < tonumber(ARGV[1]) then "
                           //如果令牌不足，返回还需等待多久时间                   
                         + "local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); "
                         + "return 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));"
                     + "else "
                           //令牌充足，往zset加数据，将String结构减去ARGV[1]                   
                         + "redis.call('zadd', permitsName, ARGV[2], struct.pack('Lc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); "
                         + "redis.call('decrby', valueName, ARGV[1]); "
                         + "return nil; "
                     + "end; "
              + "else "
                       //获取不到值，说明令牌是满的，直接操作即可                      
                     + "redis.call('set', valueName, rate); "
                     + "redis.call('zadd', permitsName, ARGV[2], struct.pack('Lc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); "
                     + "redis.call('decrby', valueName, ARGV[1]); "
                     + "return nil; "
              + "end;",
                Arrays.asList(getRawName(), getValueName(), getClientValueName(), getPermitsName(), getClientPermitsName()),
                value, System.currentTimeMillis(), random);
    }
```

1. `getRawName()`方法获取限流器的名称
2. `getValueName()`方法获取的分布式限流器值的名称
3. `getClientValueName()`方法获取的是单机限流器值的名称
4. `getPermitsName()`方法获取的是分布式zset的名称 （redisson老版本是不存在的，因为是直接对字符串的key进行操作）
5. `getClientPermitsName()`方法获取的是单机zset的名称（redisson老版本是不存在的，因为是直接对字符串的key进行操作）
5. 这里使用zset结构，因为每次请求可能需要获取不同数量的令牌，比如场景：普通用户每次请求需要2个token，但是vip用户每次请求只需要1个token

#### trySetRate方法

```java
public boolean trySetRate(RateType type, long rate, long rateInterval, RateIntervalUnit unit) {
        return get(trySetRateAsync(type, rate, rateInterval, unit));
    }

public RFuture<Boolean> trySetRateAsync(RateType type, long rate, long rateInterval, RateIntervalUnit unit) {
    	//这里就是设置上面那段lua脚本最开始获取到的限流配置
    	//type分为单机限流和分布式限流
        return commandExecutor.evalWriteNoRetryAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);"
              + "redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);"
              + "return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);",
                Collections.singletonList(getRawName()), rate, unit.toMillis(rateInterval), type.ordinal());
    }
```

#### tryAcquire(long permits, long timeout, TimeUnit unit)方法

```java
public boolean tryAcquire(long permits, long timeout, TimeUnit unit) {
    	//传递过期时间和所需令牌数
        return get(tryAcquireAsync(permits, timeout, unit));
    }

public RFuture<Boolean> tryAcquireAsync(long permits, long timeout, TimeUnit unit) {
        RPromise<Boolean> promise = new RedissonPromise<Boolean>();
        long timeoutInMillis = -1;
        if (timeout >= 0) {
            timeoutInMillis = unit.toMillis(timeout);
        }
        tryAcquireAsync(permits, promise, timeoutInMillis);
        return promise;
    }

private void tryAcquireAsync(long permits, RPromise<Boolean> promise, long timeoutInMillis) {
        long s = System.currentTimeMillis();
    	//最开始的lua脚本
        RFuture<Long> future = tryAcquireAsync(RedisCommands.EVAL_LONG, permits);
        future.onComplete((delay, e) -> {
            //异常直接结束
            if (e != null) {
                promise.tryFailure(e);
                return;
            }
            
            if (delay == null) {
                //delay为所需等待时间，如果delay为null说明限流成功
                promise.trySuccess(true);
                return;
            }
            
            if (timeoutInMillis == -1) {
                //如果timeoutInMillis为-1，表示持续获得到锁，直到成功，这里开启了一个定时任务，延迟delay秒执行
                commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                    tryAcquireAsync(permits, promise, timeoutInMillis);
                }, delay, TimeUnit.MILLISECONDS);
                return;
            }
            
            long el = System.currentTimeMillis() - s;
            long remains = timeoutInMillis - el;
            if (remains <= 0) {
                //判断剩余时间是否超时
                promise.trySuccess(false);
                return;
            }
            if (remains < delay) {
                //定时任务延迟remains返回结束
                commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                    promise.trySuccess(false);
                }, remains, TimeUnit.MILLISECONDS);
            } else {
                long start = System.currentTimeMillis();
                //剩余时间充足，开启定时任务重复请求
                commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                    long elapsed = System.currentTimeMillis() - start;
                    if (remains <= elapsed) {
                        promise.trySuccess(false);
                        return;
                    }
                    
                    tryAcquireAsync(permits, promise, remains - elapsed);
                }, delay, TimeUnit.MILLISECONDS);
            }
        });
    }
```

