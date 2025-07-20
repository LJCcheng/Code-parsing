#### Mybatis plugin

### 前言

为什么要看 `mybatis` 这部分源码？因为在日常工作中遇到一个问题，在不同项目中所用技术栈不同，导致不熟悉别人项目的 `mybatis` 插件，在使用过程中遇到分页疑问，所以花时间看了下相关源码。

### Interceptor接口

```java
public interface Interceptor {

  Object intercept(Invocation invocation) throws Throwable;

  default Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  default void setProperties(Properties properties) {
    // NOP
  }

}
```

1. `intercept`接口方法代表插件具体的执行逻辑，交由具体实现类实现
2. 默认`plugin`方法，使用 `Plugin`类进行装饰，返回一个代理类
3. 默认`setProperties`方法，进行一些属性初始化操作

### Interceptor 接口在哪里初始化的？

##### `InterceptorChain`类

```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
```

1. 属性 `interceptors` 保存所有插件
2. `pluginAll`使用插件返回代理对象
3. `addInterceptor`和`getInterceptors`方法为添加插件以及获取插件

##### Configuration 类

```java
public void addInterceptor(Interceptor interceptor) {
    interceptorChain.addInterceptor(interceptor);
  }


```

`Configuration`类添加插件，`XmlConfigBuilder`在进行调用

##### `XmlConfigBuilder` 类

```java
private void parseConfiguration(XNode root) {
    try {
      // issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }

private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        String interceptor = child.getStringAttribute("interceptor");
        Properties properties = child.getChildrenAsProperties();
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor().newInstance();
        interceptorInstance.setProperties(properties);
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
```

1. `parseConfiguration`方法为解析xml文件中的各个属性
2. `pluginElement`方法就是进行解析插件，进行循环解析每个插件

### 核心实现

##### `Intercepts`注解和`Signature`注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Intercepts {
  /**
   * Returns method signatures to intercept.
   *
   * @return method signatures
   */
  Signature[] value();
}

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Signature {
  /**
   * Returns the java type.
   *
   * @return the java type
   */
  Class<?> type();

  /**
   * Returns the method name.
   *
   * @return the method name
   */
  String method();

  /**
   * Returns java types for method argument.
   * @return java types for method argument
   */
  Class<?>[] args();
}
```

1. `Intercepts`注解起到标记插件作用
2. `Signature`注解标记插件需要代理的 **类，方法，入参**

##### `Invocation`类

```java
public class Invocation {

  private final Object target;
  private final Method method;
  private final Object[] args;

  public Invocation(Object target, Method method, Object[] args) {
    this.target = target;
    this.method = method;
    this.args = args;
  }

  public Object getTarget() {
    return target;
  }

  public Method getMethod() {
    return method;
  }

  public Object[] getArgs() {
    return args;
  }

  public Object proceed() throws InvocationTargetException, IllegalAccessException {
    return method.invoke(target, args);
  }

}
```

- 起到 保存代理 信息，包括：类，方法，入参

##### `Plugin`插件核心类

作用：返回代理对象，对mybatis方法进行拦截代理

```java
public class Plugin implements InvocationHandler {

  private final Object target;
  private final Interceptor interceptor;
  private final Map<Class<?>, Set<Method>> signatureMap;

  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }

    //Interceptor接口中默认方法进行调用，返回jdk动态代理
  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

    //jdk动态代理方法
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

    //获取Signature注解的元信息，返回map，map里面保存的是需要代理的类的方法和入参
  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<>();
    for (Signature sig : sigs) {
      Set<Method> methods = MapUtil.computeIfAbsent(signatureMap, sig.type(), k -> new HashSet<>());
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }

    //获取该类是否接口实现类，只有接口实现类才能jdk动态代理
  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[0]);
  }

}
```

1. `wrap`方法，插件的装饰方法，返回动态代理对象
2. `invoke`方法，动态代理实现的核心方法，根据 **类 和 方法** 来判断是否需要走插件逻辑
3. `getSignatureMap`方法为获取需要动态代理的 **类 和 方法**信息
4. `getAllInterfaces` 为获取当前类的接口方法，执行**`jdk`动态代理**需要



以上就是 `mybatis` 插件源码的逻辑，下面我又看了著名开源插件 `pageHelper ` 的源码

### `PageHelper` 源码学习

先介绍下包结构

- cache 包，实现了基于内存和guava的缓存，不是重点
- dialect 包，下面分了4个子包分别是：
  - auto自动注入包，根据具体的数据源选择对应的插件逻辑策略
  - helper 包，根据 `PageHelper` 实现分页 `sql` 拼接
  - rowbounds 包，根据 `rowbounds` 实现分页sql 拼接
  - replace 字符串转换
- page 包，分页参数保存
- util 工具类包

##### `PageInterceptor` 类

```java
@Intercepts(
        {
                @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
                @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
        }
)


@Override
    public Object intercept(Invocation invocation) throws Throwable {
        try {
            Object[] args = invocation.getArgs();
            MappedStatement ms = (MappedStatement) args[0];
            Object parameter = args[1];
            RowBounds rowBounds = (RowBounds) args[2];
            ResultHandler resultHandler = (ResultHandler) args[3];
            Executor executor = (Executor) invocation.getTarget();
            CacheKey cacheKey;
            BoundSql boundSql;
            //由于逻辑关系，只会进入一次
            if (args.length == 4) {
                //4 个参数时
                boundSql = ms.getBoundSql(parameter);
                cacheKey = executor.createCacheKey(ms, parameter, rowBounds, boundSql);
            } else {
                //6 个参数时
                cacheKey = (CacheKey) args[4];
                boundSql = (BoundSql) args[5];
            }
            checkDialectExists();
            //对 boundSql 的拦截处理
            if (dialect instanceof BoundSqlInterceptor.Chain) {
                boundSql = ((BoundSqlInterceptor.Chain) dialect).doBoundSql(BoundSqlInterceptor.Type.ORIGINAL, boundSql, cacheKey);
            }
            List resultList;
            //调用方法判断是否需要进行分页，如果不需要，直接返回结果
            if (!dialect.skip(ms, parameter, rowBounds)) {
                //开启debug时，输出触发当前分页执行时的PageHelper调用堆栈
                // 如果和当前调用堆栈不一致，说明在启用分页后没有消费，当前线程再次执行时消费，调用堆栈显示的方法使用不安全
                debugStackTraceLog();
                //判断是否需要进行 count 查询
                if (dialect.beforeCount(ms, parameter, rowBounds)) {
                    //查询总数
                    Long count = count(executor, ms, parameter, rowBounds, null, boundSql);
                    //处理查询总数，返回 true 时继续分页查询，false 时直接返回
                    if (!dialect.afterCount(count, parameter, rowBounds)) {
                        //当查询总数为 0 时，直接返回空的结果
                        return dialect.afterPage(new ArrayList(), parameter, rowBounds);
                    }
                }
                resultList = ExecutorUtil.pageQuery(dialect, executor,
                        ms, parameter, rowBounds, resultHandler, boundSql, cacheKey);
            } else {
                //rowBounds用参数值，不使用分页插件处理时，仍然支持默认的内存分页
                resultList = executor.query(ms, parameter, rowBounds, resultHandler, cacheKey, boundSql);
            }
            return dialect.afterPage(resultList, parameter, rowBounds);
        } finally {
            if (dialect != null) {
                dialect.afterAll();
            }
        }
    }

private Long count(Executor executor, MappedStatement ms, Object parameter,
                       RowBounds rowBounds, ResultHandler resultHandler,
                       BoundSql boundSql) throws SQLException {
        String countMsId = countMsIdGen.genCountMsId(ms, parameter, boundSql, countSuffix);
        Long count;
        //先判断是否存在手写的 count 查询
        MappedStatement countMs = ExecutorUtil.getExistedMappedStatement(ms.getConfiguration(), countMsId);
        if (countMs != null) {
            count = ExecutorUtil.executeManualCount(executor, countMs, parameter, boundSql, resultHandler);
        } else {
            if (msCountMap != null) {
                countMs = msCountMap.get(countMsId);
            }
            //自动创建
            if (countMs == null) {
                //根据当前的 ms 创建一个返回值为 Long 类型的 ms
                countMs = MSUtils.newCountMappedStatement(ms, countMsId);
                if (msCountMap != null) {
                    msCountMap.put(countMsId, countMs);
                }
            }
            count = ExecutorUtil.executeAutoCount(this.dialect, executor, countMs, parameter, boundSql, rowBounds, resultHandler);
        }
        return count;
    }
```

1. pageHelper 开源插件的注释写的很清晰了
2. 总结就是动态代理 Executor 类的 query方法，在执行的时候进行代理，动作包括：
   - 是否需要使用插件（skip方法）
   - 是否需要在分页之前先查询sql的总条数，为了解决性能问题，以及在查询 数据总数 之后是否还有必要进行分页查询等等
   - 在执行分页查询操作

##### 分页操作实现

###### `ExecutorUtil`抽象类

```java
public static <E> List<E> pageQuery(Dialect dialect, Executor executor, MappedStatement ms, Object parameter,
                                        RowBounds rowBounds, ResultHandler resultHandler,
                                        BoundSql boundSql, CacheKey cacheKey) throws SQLException {
        //判断是否需要进行分页查询
        if (dialect.beforePage(ms, parameter, rowBounds)) {
            //生成分页的缓存 key
            CacheKey pageKey = cacheKey;
            //处理参数对象
            parameter = dialect.processParameterObject(ms, parameter, boundSql, pageKey);
            //调用方言获取分页 sql
            String pageSql = dialect.getPageSql(ms, boundSql, parameter, rowBounds, pageKey);
            BoundSql pageBoundSql = new BoundSql(ms.getConfiguration(), pageSql, boundSql.getParameterMappings(), parameter);

            Map<String, Object> additionalParameters = getAdditionalParameter(boundSql);
            //设置动态参数
            for (String key : additionalParameters.keySet()) {
                pageBoundSql.setAdditionalParameter(key, additionalParameters.get(key));
            }
            //对 boundSql 的拦截处理
            if (dialect instanceof BoundSqlInterceptor.Chain) {
                pageBoundSql = ((BoundSqlInterceptor.Chain) dialect).doBoundSql(BoundSqlInterceptor.Type.PAGE_SQL, pageBoundSql, pageKey);
            }
            //执行分页查询
            return executor.query(ms, parameter, RowBounds.DEFAULT, resultHandler, pageKey, pageBoundSql);
        } else {
            //不执行分页的情况下，也不执行内存分页
            return executor.query(ms, parameter, RowBounds.DEFAULT, resultHandler, cacheKey, boundSql);
        }
    }
```

1. 先判断是否需要分页查询
2. 然后获取入参
3. 拼接分页sql
4. 动态sql参数设置
5. 执行查询

###### 分页查询sql具体实现

这里只贴oracle具体实现OracleRowBoundsDialect

```java
@Override
    public String getPageSql(String sql, RowBounds rowBounds, CacheKey pageKey) {
        int startRow = rowBounds.getOffset();
        int endRow = rowBounds.getOffset() + rowBounds.getLimit();
        StringBuilder sqlBuilder = new StringBuilder(sql.length() + 120);
        if (startRow > 0) {
            sqlBuilder.append("SELECT * FROM ( ");
        }
        if (endRow > 0) {
            sqlBuilder.append(" SELECT TMP_PAGE.*, ROWNUM PAGEHELPER_ROW_ID FROM ( ");
        }
        sqlBuilder.append("\n");
        sqlBuilder.append(sql);
        sqlBuilder.append("\n");
        if (endRow > 0) {
            sqlBuilder.append(" ) TMP_PAGE WHERE ROWNUM <= ");
            sqlBuilder.append(endRow);
            pageKey.update(endRow);
        }
        if (startRow > 0) {
            sqlBuilder.append(" ) WHERE PAGEHELPER_ROW_ID > ");
            sqlBuilder.append(startRow);
            pageKey.update(startRow);
        }
        return sqlBuilder.toString();
    }
```

