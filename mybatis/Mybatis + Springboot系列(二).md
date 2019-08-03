# Mybatis + Springboot系列(二) 
##  前文
#### 系列文章的第二篇，我们会主要重点关注一下springboot为mybatis带来的自动装配的部分，了解一下为什么这么便利。
## 正文
#### 直接看到mybatis-spring-boot-autoconfigure-2.0.0包下的spring.factories
````
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
````
#### 所以我们直接看到这个自动配置类。
````
@org.springframework.context.annotation.Configuration
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MybatisAutoConfiguration implements InitializingBean {
}
````
#### 类上面有很多注解，简单看一下几个比较重要的注解。
- @ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
	开始自动装配必须要先加载了这两个class，注意一下这两个类在mybatis中起了非常重要的作用，之后会逐步讲到
- @ConditionalOnSingleCandidate(DataSource.class)
	会加载我们之前配置的数据库相关的
- @EnableConfigurationProperties(MybatisProperties.class)
	在自动装配前，会先装配这个属性类
#### 现在我们先直接看到 SqlSessionFactory和SqlSessionFactoryBean这两个类的注入。
````

  @Bean
  @ConditionalOnMissingBean
  public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
  		// 创建一个FactoryBean类
	    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
	    factory.setDataSource(dataSource);
	    factory.setVfs(SpringBootVFS.class);
	    
	    if (StringUtils.hasText(this.properties.getConfigLocation())) {
	      factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
	    }
	    // 注入configuration类
	    applyConfiguration(factory);
	   	...
	   	// 设置各类属性
	   	...
	    return factory.getObject();
  }
  
  private void applyConfiguration(SqlSessionFactoryBean factory) {
    Configuration configuration = this.properties.getConfiguration();
    if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
      configuration = new Configuration();
    }
    if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
      for (ConfigurationCustomizer customizer : this.configurationCustomizers) {
        customizer.customize(configuration);
      }
    }
    factory.setConfiguration(configuration);
  }

````

#### 看到上述代码，会得到一个注入了各个参数的SqlSessionFactoryBean，然后会调用factory.getObject方法,继续跟到里面看看做了什么

````
  @Override
  public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
      afterPropertiesSet();
    }

    return this.sqlSessionFactory;
  }
  
  @Override
  public void afterPropertiesSet() throws Exception {
    notNull(dataSource, "Property 'dataSource' is required");
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
              "Property 'configuration' and 'configLocation' can not specified with together");

    this.sqlSessionFactory = buildSqlSessionFactory();
  }
````

#### 可以看到最后会返回一个sqlSessionFactory，接下来要看的这个buildSqlSessionFactory方法非常重要，我们配置的mapper都会在这里被加载。
````
  protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
    final Configuration targetConfiguration;

    XMLConfigBuilder xmlConfigBuilder = null;
    if (this.configuration != null) {
      targetConfiguration = this.configuration;
      if (targetConfiguration.getVariables() == null) {
        targetConfiguration.setVariables(this.configurationProperties);
      } else if (this.configurationProperties != null) {
        targetConfiguration.getVariables().putAll(this.configurationProperties);
      }
    } else if (this.configLocation != null) {
      xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
      targetConfiguration = xmlConfigBuilder.getConfiguration();
    } else {
      LOGGER.debug(() -> "Property 'configuration' or 'configLocation' not specified, using default MyBatis Configuration");
      targetConfiguration = new Configuration();
      Optional.ofNullable(this.configurationProperties).ifPresent(targetConfiguration::setVariables);
    }
````
#### 这段代码主要是要验证一下configuration类是否已经创建，如果不是则需要创建。
````
	if (hasLength(this.typeAliasesPackage)) {
      ...
    }

    if (!isEmpty(this.typeAliases)) {
      ...
    }

    if (!isEmpty(this.plugins)) {
      ...
    }

    if (hasLength(this.typeHandlersPackage)) {
      ...
    }

    if (!isEmpty(this.typeHandlers)) {
      ...
    }

    if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
      ...
    }
    if (xmlConfigBuilder != null) {
      ...
    }

    targetConfiguration.setEnvironment(new Environment(this.environment,
        this.transactionFactory == null ? new SpringManagedTransactionFactory() : this.transactionFactory,
        this.dataSource));

    if (!isEmpty(this.mapperLocations)) {
      ...
    } else {
      LOGGER.debug(() -> "Property 'mapperLocations' was not specified or no matching resources found");
    }
````
#### 主要判断几个参数是否存在，这边列一下几个参数以及作用
- typeAliasesPackage
- typeAliases
- plugins
- typeHandlersPackage
- typeHandlers
- databaseIdProvider
- xmlConfigBuilder
- mapperLocations

#### 全部校验过参数之后，开始build
```
    return this.sqlSessionFactoryBuilder.build(targetConfiguration);
    
    public SqlSessionFactory build(Configuration config) {
    	return new DefaultSqlSessionFactory(config);
  	 }
```

#### 可以看到最终会返回一个DefaultSqlSessionFactory，点开这个类会看到里面的方法全都是openSession的方法，这个也是后面文章的重点。现在先回到自动配置类中，看到另一个类的bean的生成。
````
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
      return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
      return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
````

#### 看到这个方法的入参，就是我们创建的DefaultSqlSessionFactory。看到又出现了一个配置参数，executorType,这个参数是一个枚举类，总共有以下三种选择，如果我们没有配置，默认会选择Simple。
````
public enum ExecutorType {
  SIMPLE, REUSE, BATCH
}
````
#### 再继续往下看
````
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
    this(sqlSessionFactory, executorType,
        new MyBatisExceptionTranslator(
            sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(), true));
  }
  
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
````
#### 上面几行进行了参数的校验，同时对参数进行了配置，看到最后一句。看到这种写法我们应该直接想到这事jdk自带的动态代理的语法糖写法，作为行为加强用，所以我们可以直接看一下SqlSessionInterceptor这个类中做了什么事。
````
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        ...
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
````
#### 可以看到这个sqlSessionProxy在执行的时候，会被动态代理在真正执行方法的前后增强getSqlSession和判断是否需要对事务commit的操作。其中这个getSqlSession方法在后面也会对里面的逻辑认真分析一下。
#### 通过上面的文章，我们分析了一下sqlSessionTemplate和sqlSessionFactory两个bean被创建的过程，虽然之前说了这两个类非常重要，但是我们还是没看到mybatis怎么使用这两个类的，以及我们写的mapper文件怎么被用起来。由于篇幅限制，这部分内容我会在下一篇文章中讲到，重点会讲一下我们写的mapper文件是怎么被用起来的，大家可以先回想一下@MapperScan和@Mapper这两个注解我们是在什么时候用到的。