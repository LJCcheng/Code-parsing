## 重试

实现主要运用函数式编程

这里类主要由一下属性构成

```java
	//请求超时时间
	private AttemptTimeLimiter<V> attemptTimeLimiter;
	//暂停重试的策略，包含了3种策略 1.从不停止策略 2. 重试几次停止 3. 延迟多久停止
    private StopStrategy stopStrategy;
	//重试间隔时间策略，包含了7种策略 1. 随机间隔 2. 固定间隔 3. 随着时间逐渐增加时间间隔 4.指数级的增长时间间隔
	//5. 斐波那契数列计算时间间隔 6. 多个策略相加形成间隔时间 7. 发生异常的情况
    private WaitStrategy waitStrategy;
	//睡眠阻塞策略，线程睡眠多久
    private BlockStrategy blockStrategy;
	//是否需要重试
    private Predicate<Attempt<V>> rejectionPredicate = Predicates.alwaysFalse();
	//重试监听器，可以用于重试的时候执行其他功能
    private List<RetryListener> listeners = new ArrayList<RetryListener>();

	public static <V> RetryerBuilder<V> newBuilder() {
        return new RetryerBuilder<V>();
    }
```

#### 设置策略

```java
//设置重试监听器
public RetryerBuilder<V> withRetryListener(@Nonnull RetryListener listener) {
        Preconditions.checkNotNull(listener, "listener may not be null");
        listeners.add(listener);
        return this;
    }

    /**
     * Sets the wait strategy used to decide how long to sleep between failed attempts.
     * The default strategy is to retry immediately after a failed attempt.
     *
     * @param waitStrategy the strategy used to sleep between failed attempts
     * @return <code>this</code>
     * @throws IllegalStateException if a wait strategy has already been set.
     */
	//设置重试间隔等待策略
    public RetryerBuilder<V> withWaitStrategy(@Nonnull WaitStrategy waitStrategy) throws IllegalStateException {
        Preconditions.checkNotNull(waitStrategy, "waitStrategy may not be null");
        Preconditions.checkState(this.waitStrategy == null, "a wait strategy has already been set %s", this.waitStrategy);
        this.waitStrategy = waitStrategy;
        return this;
    }

    /**
     * Sets the stop strategy used to decide when to stop retrying. The default strategy is to not stop at all .
     *
     * @param stopStrategy the strategy used to decide when to stop retrying
     * @return <code>this</code>
     * @throws IllegalStateException if a stop strategy has already been set.
     */
	//停止重试策略
    public RetryerBuilder<V> withStopStrategy(@Nonnull StopStrategy stopStrategy) throws IllegalStateException {
        Preconditions.checkNotNull(stopStrategy, "stopStrategy may not be null");
        Preconditions.checkState(this.stopStrategy == null, "a stop strategy has already been set %s", this.stopStrategy);
        this.stopStrategy = stopStrategy;
        return this;
    }


    /**
     * Sets the block strategy used to decide how to block between retry attempts. The default strategy is to use Thread#sleep().
     *
     * @param blockStrategy the strategy used to decide how to block between retry attempts
     * @return <code>this</code>
     * @throws IllegalStateException if a block strategy has already been set.
     */
	//阻塞睡眠策略
    public RetryerBuilder<V> withBlockStrategy(@Nonnull BlockStrategy blockStrategy) throws IllegalStateException {
        Preconditions.checkNotNull(blockStrategy, "blockStrategy may not be null");
        Preconditions.checkState(this.blockStrategy == null, "a block strategy has already been set %s", this.blockStrategy);
        this.blockStrategy = blockStrategy;
        return this;
    }


    /**
     * Configures the retryer to limit the duration of any particular attempt by the given duration.
     *
     * @param attemptTimeLimiter to apply to each attempt
     * @return <code>this</code>
     */
	//请求超时时间
    public RetryerBuilder<V> withAttemptTimeLimiter(@Nonnull AttemptTimeLimiter<V> attemptTimeLimiter) {
        Preconditions.checkNotNull(attemptTimeLimiter);
        this.attemptTimeLimiter = attemptTimeLimiter;
        return this;
    }
```

#### 设置重试条件

```java
//所有异常重试
public RetryerBuilder<V> retryIfException() {
        rejectionPredicate = Predicates.or(rejectionPredicate, new ExceptionClassPredicate<V>(Exception.class));
        return this;
    }

    /**
     * Configures the retryer to retry if a runtime exception (i.e. any <code>RuntimeException</code> or subclass
     * of <code>RuntimeException</code>) is thrown by the call.
     *
     * @return <code>this</code>
     */
	//runtimeException才重试
    public RetryerBuilder<V> retryIfRuntimeException() {
        rejectionPredicate = Predicates.or(rejectionPredicate, new ExceptionClassPredicate<V>(RuntimeException.class));
        return this;
    }

    /**
     * Configures the retryer to retry if an exception of the given class (or subclass of the given class) is
     * thrown by the call.
     *
     * @param exceptionClass the type of the exception which should cause the retryer to retry
     * @return <code>this</code>
     */
	//指定类型异常才重试
    public RetryerBuilder<V> retryIfExceptionOfType(@Nonnull Class<? extends Throwable> exceptionClass) {
        Preconditions.checkNotNull(exceptionClass, "exceptionClass may not be null");
        rejectionPredicate = Predicates.or(rejectionPredicate, new ExceptionClassPredicate<V>(exceptionClass));
        return this;
    }

    /**
     * Configures the retryer to retry if an exception satisfying the given predicate is
     * thrown by the call.
     *
     * @param exceptionPredicate the predicate which causes a retry if satisfied
     * @return <code>this</code>
     */
	//异常满足条件才重试
    public RetryerBuilder<V> retryIfException(@Nonnull Predicate<Throwable> exceptionPredicate) {
        Preconditions.checkNotNull(exceptionPredicate, "exceptionPredicate may not be null");
        rejectionPredicate = Predicates.or(rejectionPredicate, new ExceptionPredicate<V>(exceptionPredicate));
        return this;
    }

    /**
     * Configures the retryer to retry if the result satisfies the given predicate.
     *
     * @param resultPredicate a predicate applied to the result, and which causes the retryer
     *                        to retry if the predicate is satisfied
     * @return <code>this</code>
     */
	//结果满足条件才重试
    public RetryerBuilder<V> retryIfResult(@Nonnull Predicate<V> resultPredicate) {
        Preconditions.checkNotNull(resultPredicate, "resultPredicate may not be null");
        rejectionPredicate = Predicates.or(rejectionPredicate, new ResultPredicate<V>(resultPredicate));
        return this;
    }
```

#### 建造者设计

```java
public Retryer<V> build() {
        AttemptTimeLimiter<V> theAttemptTimeLimiter = attemptTimeLimiter == null ? AttemptTimeLimiters.<V>noTimeLimit() : attemptTimeLimiter;
        StopStrategy theStopStrategy = stopStrategy == null ? StopStrategies.neverStop() : stopStrategy;
        WaitStrategy theWaitStrategy = waitStrategy == null ? WaitStrategies.noWait() : waitStrategy;
        BlockStrategy theBlockStrategy = blockStrategy == null ? BlockStrategies.threadSleepStrategy() : blockStrategy;

    	//返回Retryer，具体重试方法就是它实现的
        return new Retryer<V>(theAttemptTimeLimiter, theStopStrategy, theWaitStrategy, theBlockStrategy, rejectionPredicate, listeners);
    }
```

#### 重试条件类

```java
//异常
private static final class ExceptionClassPredicate<V> implements Predicate<Attempt<V>> {

        private Class<? extends Throwable> exceptionClass;

        public ExceptionClassPredicate(Class<? extends Throwable> exceptionClass) {
            this.exceptionClass = exceptionClass;
        }

        @Override
        public boolean apply(Attempt<V> attempt) {
            if (!attempt.hasException()) {
                return false;
            }
            return exceptionClass.isAssignableFrom(attempt.getExceptionCause().getClass());
        }
    }

	//结果
    private static final class ResultPredicate<V> implements Predicate<Attempt<V>> {

        private Predicate<V> delegate;

        public ResultPredicate(Predicate<V> delegate) {
            this.delegate = delegate;
        }

        @Override
        public boolean apply(Attempt<V> attempt) {
            if (!attempt.hasResult()) {
                return false;
            }
            V result = attempt.getResult();
            return delegate.apply(result);
        }
    }

	//异常
    private static final class ExceptionPredicate<V> implements Predicate<Attempt<V>> {

        private Predicate<Throwable> delegate;

        public ExceptionPredicate(Predicate<Throwable> delegate) {
            this.delegate = delegate;
        }

        @Override
        public boolean apply(Attempt<V> attempt) {
            if (!attempt.hasException()) {
                return false;
            }
            return delegate.apply(attempt.getExceptionCause());
        }
    }
```

#### Retryer类

重试的主要逻辑，将各种策略组合起来

```java
public V call(Callable<V> callable) throws ExecutionException, RetryException {
        long startTime = System.nanoTime();
    	//遍历重试
        for (int attemptNumber = 1; ; attemptNumber++) {
            Attempt<V> attempt;
            try {
                //执行callable函数，这里attemptTimeLimiter接口有两个实现类，一个是不设置超时时间，另一个是设置超时时间
                V result = attemptTimeLimiter.call(callable);
                //封装结果
                attempt = new ResultAttempt<V>(result, attemptNumber, TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime));
            } catch (Throwable t) {
                //封装异常
                attempt = new ExceptionAttempt<V>(t, attemptNumber, TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime));
            }

            //如果有监听器，执行监听器中的方法
            //拓展：这里我们可以自己自定义监听器，执行一些特殊逻辑
            for (RetryListener listener : listeners) {
                listener.onRetry(attempt);
            }
			//是否需要重试，不需要重试就结束返回
            if (!rejectionPredicate.apply(attempt)) {
                return attempt.get();
            }
            //停止策略判断是否需要停止
            if (stopStrategy.shouldStop(attempt)) {
                throw new RetryException(attemptNumber, attempt);
            } else {
                //计算重试间隔的时间
                long sleepTime = waitStrategy.computeSleepTime(attempt);
                try {
                    //线程睡眠
                    blockStrategy.block(sleepTime);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw new RetryException(attemptNumber, attempt);
                }
            }
        }
    }
```

#### Attempt接口

由它的两个子类来封装结果，包括正常返回结果和异常

```java
//封装正常结果
static final class ResultAttempt<R> implements Attempt<R> {
        private final R result;
        private final long attemptNumber;
        private final long delaySinceFirstAttempt;

        public ResultAttempt(R result, long attemptNumber, long delaySinceFirstAttempt) {
            this.result = result;
            this.attemptNumber = attemptNumber;
            this.delaySinceFirstAttempt = delaySinceFirstAttempt;
        }

        @Override
        public R get() throws ExecutionException {
            return result;
        }

        @Override
        public boolean hasResult() {
            return true;
        }

        @Override
        public boolean hasException() {
            return false;
        }

        @Override
        public R getResult() throws IllegalStateException {
            return result;
        }

        @Override
        public Throwable getExceptionCause() throws IllegalStateException {
            throw new IllegalStateException("The attempt resulted in a result, not in an exception");
        }

        @Override
        public long getAttemptNumber() {
            return attemptNumber;
        }

        @Override
        public long getDelaySinceFirstAttempt() {
            return delaySinceFirstAttempt;
        }
    }

static final class ExceptionAttempt<R> implements Attempt<R> {
        private final ExecutionException e;
        private final long attemptNumber;
        private final long delaySinceFirstAttempt;

        public ExceptionAttempt(Throwable cause, long attemptNumber, long delaySinceFirstAttempt) {
            this.e = new ExecutionException(cause);
            this.attemptNumber = attemptNumber;
            this.delaySinceFirstAttempt = delaySinceFirstAttempt;
        }

        @Override
        public R get() throws ExecutionException {
            throw e;
        }

        @Override
        public boolean hasResult() {
            return false;
        }

        @Override
        public boolean hasException() {
            return true;
        }

        @Override
        public R getResult() throws IllegalStateException {
            throw new IllegalStateException("The attempt resulted in an exception, not in a result");
        }

        @Override
        public Throwable getExceptionCause() throws IllegalStateException {
            return e.getCause();
        }

        @Override
        public long getAttemptNumber() {
            return attemptNumber;
        }

        @Override
        public long getDelaySinceFirstAttempt() {
            return delaySinceFirstAttempt;
        }
    }
```

#### 超时请求实现

```java
private static final class FixedAttemptTimeLimit<V> implements AttemptTimeLimiter<V> {

    	//超时请求具体实现
        private final TimeLimiter timeLimiter;
    	//超时时间
        private final long duration;
    	//超时单位
        private final TimeUnit timeUnit;

        public FixedAttemptTimeLimit(long duration, @Nonnull TimeUnit timeUnit) {
            this(new SimpleTimeLimiter(), duration, timeUnit);
        }

        public FixedAttemptTimeLimit(long duration, @Nonnull TimeUnit timeUnit, @Nonnull ExecutorService executorService) {
            this(new SimpleTimeLimiter(executorService), duration, timeUnit);
        }

        private FixedAttemptTimeLimit(@Nonnull TimeLimiter timeLimiter, long duration, @Nonnull TimeUnit timeUnit) {
            Preconditions.checkNotNull(timeLimiter);
            Preconditions.checkNotNull(timeUnit);
            this.timeLimiter = timeLimiter;
            this.duration = duration;
            this.timeUnit = timeUnit;
        }

        @Override
        public V call(Callable<V> callable) throws Exception {
            return timeLimiter.callWithTimeout(callable, duration, timeUnit, true);
        }
    }

public <T extends @Nullable Object> T callWithTimeout(
      Callable<T> callable, long timeoutDuration, TimeUnit timeoutUnit)
      throws TimeoutException, InterruptedException, ExecutionException {
    checkNotNull(callable);
    checkNotNull(timeoutUnit);
    checkPositiveTimeout(timeoutDuration);
	//线程池提交
    Future<T> future = executor.submit(callable);

    try {
        //通过future实现超时请求
      return future.get(timeoutDuration, timeoutUnit);
    } catch (InterruptedException | TimeoutException e) {
      future.cancel(true /* mayInterruptIfRunning */);
      throw e;
    } catch (ExecutionException e) {
      wrapAndThrowExecutionExceptionOrError(e.getCause());
      throw new AssertionError();
    }
  }
```

