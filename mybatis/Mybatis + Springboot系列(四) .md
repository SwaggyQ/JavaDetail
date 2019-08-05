# Mybatis + Springboot系列(四) 
## 前文
#### 至此我们已经在项目中注入了这个mapperProxy类了，本文接着看看具体调用sql的时候发生了什么
## 正文
#### 用我们一开始的例子引出我们现在的内容
````
public interface UserMapper {

    @Select("SELECT * FROM entity limit #{count}")
    @Results({
            @Result(property = "id",  column = "user_sex"),
            @Result(property = "entityCode", column = "entity_code"),
            @Result(property = "entityTypeId",  column = "entity_type_id"),
    })
    List<Entity> getAll(@Param("count") Integer count);
}

public class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void testQuery() throws Exception {
        List<Entity> entities = new LinkedList<>();
        entities = userMapper.getAll(1);
        System.out.println(entities);
    }

}
````
#### 现在我们已经在Test类中得到了UserMapper类，现在准备调用userMapper.getAll(1)方法，因为我们知道我们现在真正操作的类是mapperProxy代理的mapper类，由代理模式的原理可得，我们现在会进入到代理类的invoke方法中。
````
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // 会有一个缓存的功能，未存在则需要新建
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 执行对应的方法
    return mapperMethod.execute(sqlSession, args);
  }
````

#### 终于讲到最重要的执行方法的阶段了，在真正执行前我们还要先知道这个mapperMethod是什么，以及在里面有什么属性
````
public class MapperMethod {

  private final SqlCommand command;
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
}

public static class SqlCommand {

    private final String name;
    private final SqlCommandType type;

    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      ...
    }
}

public static class MethodSignature {

    private final boolean returnsMany;
    private final boolean returnsMap;
    private final boolean returnsVoid;
    private final boolean returnsCursor;
    private final boolean returnsOptional;
    private final Class<?> returnType;
    private final String mapKey;
    private final Integer resultHandlerIndex;
    private final Integer rowBoundsIndex;
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      ...
    }   
}
````

#### 看到这个MapperMethod里面包装了两个重要的类，分别是MethodSignature和SqlCommand，这里面包装了所有我们写的sql相关的参数和返回参数以及语句种类的参数，里面的参数我们先这边跳过不说，之后再详细说一下这个Configuration类的相关东西，我们先继续往下看，看到后面的execute方法。

````
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    // 选择语句的类型，可以看到不管是什么类型，最终还是会落回到sqlSession，我们这边继续之前的select方法
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
        	// 按照之前的例子会进入到这个分支，每个分支都是可以类比的
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional() &&
              (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
  
  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    // 参数转换
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
````

#### 所有不同的语句种类，不同的语句参数最终都会指向一个sqlSession的方法，而这个sqlSession的具体实现类也就是我们之前创建的SqlSessionTemplate类，我们随便找一个sqlSession.selectList()方法作为例子，看看后面的实际逻辑。
````
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }

  @Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.sqlSessionProxy.selectList(statement, parameter);
  }
````

#### 又是熟悉的proxy模式，直接看到SqlSessionInterceptor增强类中的内容。
````
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 可以看到每次执行前都会先去getSqlSession，里面其实是一个threadLocal，本文中对这个方法也先跳过
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
      	 // 真正的执行方法
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          // 判断是否需要执行一次事务提交
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        ...
      }
    }
  }
````
#### 看到在执行方法的前后做了操作之后，最终还是回到了执行method方法。
````
  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      // 最后可以看到会由一个executor来执行最终的方法
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
````
#### 我们会看到最终会由一个executor的类来执行最后的操作，这个类的创建是由Configuration来生成的，这个配置类在整个mybatis的流程中都扮演了非常重要的角色，我们在后面会专门用一篇文章来描述，我们这边先直接看到里面的方法。
````
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    // 总共有以下几种类型
    - Batch
    - Reuse
    - Simple
    - Caching
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    // 是否包装一层缓存
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
````
#### 我们在本文中的例子会用Caching + Simple的组合，现在我们看到query方法
````
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  	// 最后真正执行的sql的载体
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    // 获取唯一的缓存key
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    // 真正的query方法
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
      // 得到缓存
    Cache cache = ms.getCache();
    if (cache != null) {
    	// 是否需要清空缓存
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
      	// 不支持缓存输出类型的sql
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        // 从真正管理缓存的tcm中获取缓存结果
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
        	// 如果没有在缓存中得到结果，则需要新的读取并放入缓存，这里的delegate.query其实就是调用SimpleExecutor的方法
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
  
  
  // BaseExecutor.java
  @SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // 
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
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
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
````