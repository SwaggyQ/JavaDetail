# Mybatis + Springboot系列(六) 
## 前文
#### 这篇文章我们会讲一讲mybatis涉及到的关于缓存的事情。在Mybatis中有两种缓存的方式，一般来说就是一级缓存和二级缓存，本文就会针对这两个类型来进行解析。
## 正文
#### 直接看到一级缓存是什么，回想我们之前在执行一个sql的时候，会拿到一个sqlSession，然后由sqlSession里面的Executor来执行真正的操作，而我们今天说的缓存也就是针对这一步执行的。先看到BaseExecutor的query方法。
````
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    // 缓存Key，用于判断是否可以直接缓存中读取结果的key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
 }
 
  @SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ...
    List<E> list;
    try {
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    ...
    return list;
  }
````
#### 从上面两个方法，很清楚的看到，通过构建了一个CacheKey之后，在真正查询之前，会从localCache中通过CacheKey来获取一次结果，再深入进去可以看到这个localCache其实内部是一个HashMap，根据HashMap的特性，我们直接看一下CacheKey的equal和hashcode方法，看看跟什么有关
````
  @Override
  public boolean equals(Object object) {
    if (this == object) {
      return true;
    }
    if (!(object instanceof CacheKey)) {
      return false;
    }

    final CacheKey cacheKey = (CacheKey) object;

    if (hashcode != cacheKey.hashcode) {
      return false;
    }
    if (checksum != cacheKey.checksum) {
      return false;
    }
    if (count != cacheKey.count) {
      return false;
    }

    for (int i = 0; i < updateList.size(); i++) {
      Object thisObject = updateList.get(i);
      Object thatObject = cacheKey.updateList.get(i);
      if (!ArrayUtil.equals(thisObject, thatObject)) {
        return false;
      }
    }
    return true;
  }

  @Override
  public int hashCode() {
    return hashcode;
  }
````
#### 可以看到跟hashcode，checksum，count还有跟updateList有关，那再看看这几个参数是什么时候改变的。
````
  public CacheKey() {
    this.hashcode = DEFAULT_HASHCODE;    // 默认17
    this.multiplier = DEFAULT_MULTIPLYER;     // 默认37
    this.count = 0;
    this.updateList = new ArrayList<>();
  }

  public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);

    count++;
    checksum += baseHashCode;
    baseHashCode *= count;

    hashcode = multiplier * hashcode + baseHashCode;

    updateList.add(object);
  }
````
#### 由上可知，对于两个CacheKey是否相等的判断跟update方法有很大关系，甚至对update不同Object的顺序也有要求。现在看到在BaseExecutor中创建CacheKey的地方。
````
@Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    // 跟下面的几个参数都有关系，其中id代表
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        // 对每个mapping的值都有要求
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      // 对环境也需要考虑
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
````
#### 这边借用一下<五月的仓颉>老师的总结
> 到了这里应当一目了然了，MyBastis从四组共五个条件判断两次查询是相同的：

-  <select 标签所在的Mapper的Namespace+<select 标签的id属性
- RowBounds的offset和limit属性，RowBounds是MyBatis用于处理分页的一个类，offset默认为0，limit默认为Integer.MAX_VALUE
- <select 标签中定义的sql语句
- 输入参数的具体参数值，一个int值就update一个int，一个String值就update一个String，一个List就轮询里面的每个元素进行update


#### 以上就是在两次调用select的时候，一级缓存判断两个sql是否相同，以及是否可以直接从缓存中获取结果，从而减小对数据库的查询压力。同时我们可以看到，这个localCache其实是一个PerpetualCache类型的缓存类型，同时由于这个localCache属于BaseExecutor的一个属性，同时BaseExecutor属于sqlSession，所以我们说的一级缓存说的都是以SqlSession为维度的。
#### 接下来我们会继续看到一个二级缓存，这个二级缓存是以mapper为维度的。讲二级缓存我们就要重新回到我们之前说的CachingExecutor的query方法中。
````
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    // 从MapperStatement中得到cache，所以我们说二级缓存是以mapper为维度的
    Cache cache = ms.getCache();
    if (cache != null) {
    	// 判断是否需要刷新cache
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        // 和一级缓存不同的是，二级缓存是由tcm管理的
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
````
