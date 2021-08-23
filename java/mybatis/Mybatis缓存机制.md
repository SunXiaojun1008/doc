## Mybatis缓存详解

### Cache缓存

缓存一般是ORM框架都会提供的功能，目的是提升查询执行的效率以及减少与数据库的交互，降低数据库的压力。跟Hibernate一样，Mybatis也提供了缓存功能，一级缓存和二级缓存，并且预留了集成第三方缓存的接口。

### 缓存体系结构

- Mybatis与缓存相关的类都在cache包内，其中有一个Cache接口，只有一个默认实现类`PerpetualCache`，内部使用`HashMap`存储数据。

```java
public class PerpetualCache implements Cache {
  private final String id;
  private final Map<Object, Object> cache = new HashMap<>();
  ...
}
```

- PerpetualCache这个对象一定会创建，所以这个叫做基础缓存。但是缓存又可以又很多额外功能，比如回收策略、日志记录、定时刷新等等，这些功能根据具体需求可以进行增删。

- 除了基础缓存之外，Mybatis也定义了很多的装饰器，同样实现了Cache接口，通过这些装饰器可以额外实现很多的功能。

  > 装饰器模式：在不改变原有对象的基础上，将功能附加到对象上，提供了比继承更有弹性的替代方案（扩展原有对象的功能）

![Cache](..\..\images\mybatis\Cache.png)

- 当然不管经过多少层装饰，最后使用的还是基本的实现类（PerpetualCache）。

- 所有的缓存实现类大致可以分为三类：基本缓存、淘汰算法缓存、装饰器缓存。

### 一级缓存

一级缓存也叫本地缓存（Local Cache），Mybatis的一级缓存是在会话（SqlSession）层面进行缓存的。Mybatis的一级缓存是默认开启的，不需要任何的配置，若需要关闭一级缓存需要设置`localCacheScope = STATEMENT`。

```java
// org.apache.ibatis.executor.BaseExecutor.java
@SuppressWarnings("unchecked")
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    // 异常体系之 ErrorContext
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      // flushCache="true"时，即使是查询，也清空一级缓存
      clearLocalCache();
    }
    List<E> list;
    try {
      // 防止递归查询重复处理缓存
      queryStack++;
      // 查询一级缓存
      // ResultHandler 和 ResultSetHandler的区别
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 真正的查询流程
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      // 当LocalCacheScope  == LocalCacheScope.STATEMENT ，清空缓存即不使用缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```

首先我们必须弄清楚一个问题，在Mybatis执行的流程里面，涉及到这么多的对象，那么缓存PerpetualCache应该放在哪个对象里去维护？

上面说到过，一级缓存是在会话层面进行缓存，作为SqlSession的一个属性，与SqlSession共存亡，这样就不需要为SqlSession进行编号，再根据SqlSession的编号去查询对应的缓存了。

- 在DefaultSqlSession中只有两个对象：Configuration 和Executor。Configuration 是全局的，不属于SqlSession，所以缓存只可能存在Executor里面维护，实际上它是在基本执行器SimpleExecutor/ReuseExecutor/BatchExecutor的父类BaseExecutor的构造函数中持有了PerpetualCache。

- 在同一个会话中，多次执行相同的SQL语句，会直接从内存取到缓存的结果，不会再次发送SQL到数据库。但是不同的会话中，即使执行的SQL一模一样（通过一个Mapper的同一个方法的相同参数调用），也不可能使用到一级缓存。

![一级缓存](..\..\images\mybatis\一级缓存.png)

- 当在同一个会话中，执行Update或者Delete语句时，会导致一级缓存被清空

**一级缓存的不足**

- 使用一级缓存时，会导致查询到过期数据。

> 因为一级缓存不能跨会话共享，在不同会话之间可能存在不同的缓存。当会话一查询数据后，缓存在本地，此时会话二修改了这条数据，且会话一还未中断，这个时候如果会话一再次执行查询，则因为是同一个会话，同样的SQL，会直接从缓存中读取数据，此时的数据与数据库中实际数据不同。

### 二级缓存

- 二级缓存是用来解决一级缓存不能跨会话的问题，范围是namespace级别的，可以被多个SqlSession共享（只要是同一个接口里面的相同方法，都可以共享），生命周期和应用同步。由此可以推荐，Mybatis在执行查询时，会优先查询二级缓存的数据。

- 因为二级缓存被多个SqlSession共享，那SqlSession本身和它里面的BaseExecutor已经不满足需求了，这个时候我们需要在BaseExecutor之外创建一个对象。同时，二级缓存是选择性开启的，也就说开启二级缓存就启用这个对象，如果没有就不使用这个对象。

- Mybatis中用了一个装饰器的类来维护`CachingExecutor`，如果启用二级缓存，Mybatis在创建Executor对象的时候，会对Executor进行装饰。

  ```java
  // DefaultSqlSessionFactory.java 
  // 根据事务工厂和默认的执行器类型，创建执行器 >>
  final Executor executor = configuration.newExecutor(tx, execType);
  ```

  ```java
  // Configuration.java
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
      executorType = executorType == null ? defaultExecutorType : executorType;
      executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
      Executor executor;
      if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
      } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
      } else {
        // 默认 SimpleExecutor
        executor = new SimpleExecutor(this, transaction);
      }
      // 二级缓存开关，settings 中的 cacheEnabled 默认是 true
      if (cacheEnabled) {
        executor = new CachingExecutor(executor);
      }
      // 植入插件的逻辑，至此，四大对象已经全部拦截完毕
      executor = (Executor) interceptorChain.pluginAll(executor);
      return executor;
    }
  ```

- CachingExecutor对于查询请求,会判断二级缓存是否有缓存结果，如果有就直接返回，如果没有委派给真正的查询器Executor实现类，比如SimpleExecutor来执行查询，再走到一级缓存的流程。最后会把结果缓存起来，并返回给用户。

![二级缓存](..\..\\images\mybatis\二级缓存.png)

- 二级缓存通过`cacheEnabled`进行开关，默认为`true`即二级缓存默认也是开启的。但是每个Mapper的二级缓存开关是默认关闭的，一个Mapper需要使用二级缓存，还需要单独打开。

  ```xml
  <!-- 声明这个namespace使用二级缓存 -->
  <cache type="org.apache.ibatis.cache.impl.PerpetualCache"
         size="1024" <!-- 最多缓存对象个数，默认1024 -->
         eviction="LRU" <!-- 回收策略 LRU-->
         flushInterval="120000" <!-- 自动刷新时间ms，未配置时，只有调用时才刷新 -->
         readOnly="false"/> <!-- 默认是false（安全），改为true可读写时，对象必须支持序列化 --> 
  ```

  

