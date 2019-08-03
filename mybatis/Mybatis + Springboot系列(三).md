# Mybatis + Springboot系列(三)  
## 前文
#### 前面的文章我们大致看了一下自动装配类里面为我们做了什么事，但是也留了一个疑问，因为我们还没有看到我们写的mapper类的东西在哪里被用起来，这就是本文想要去解释的东西。
## 正文
#### 回想我们第一篇文章中简单提到过@MapperScan和@Mapper两个注解，当时说的是前者放在springboot启动类上，扫描全部的mapper类，后者是一个个放在单独的mapper类上，那这两个注解到底做了什么呢？
#### 看一下@Mapper注解
````
@Documented
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD, ElementType.FIELD, ElementType.PARAMETER })
public @interface Mapper {
  // Interface Mapper
}
````
#### 看到这个注解里面没写什么重要的东西，这是因为重要的逻辑还是写在了那个自动配置类里。
````
  @org.springframework.context.annotation.Configuration
  @Import({ AutoConfiguredMapperScannerRegistrar.class })
  @ConditionalOnMissingBean(MapperFactoryBean.class)
  public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
      logger.debug("No {} found.", MapperFactoryBean.class.getName());
    }
  }
````
#### 重点都在那个import的类中
````
  public static class AutoConfiguredMapperScannerRegistrar
      implements BeanFactoryAware, ImportBeanDefinitionRegistrar, ResourceLoaderAware {

    private BeanFactory beanFactory;

    private ResourceLoader resourceLoader;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		...
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
      this.beanFactory = beanFactory;
    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
      this.resourceLoader = resourceLoader;
    }
  }
````
#### 可以看到这个注册类是实现了ImportBeanDefinitionRegistrar接口，其他的调用逻辑会交由springboot框架来完成，所以我们只需要重点看到这个registerBeanDefinitions方法，由这个方法会扫描所有@Mapper注解的类。
````
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      if (!AutoConfigurationPackages.has(this.beanFactory)) {
        logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.");
        return;
      }

      logger.debug("Searching for mappers annotated with @Mapper");
		// 得到springboot启动类所在的目录
      List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
      if (logger.isDebugEnabled()) {
        packages.forEach(pkg -> logger.debug("Using auto-configuration base package '{}'", pkg));
      }
	// 构建一个ClassPathMapperScanner类，就是扫描Mapper文件的主要逻辑所在
      ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
      if (this.resourceLoader != null) {
        scanner.setResourceLoader(this.resourceLoader);
      }
      // 扫描加了Mapper注解的类
      scanner.setAnnotationClass(Mapper.class);
      scanner.registerFilters();
      // 扫描对应的目录，也就是springboot启动类所在的目录
      scanner.doScan(StringUtils.toStringArray(packages));
    }
````
#### 好了，一步步引导我们来到了这个最关键的类了，我们写的所有的Mapper被扫描注册以及之后被注入的关键所在。
````
// 先可以看到这个类是继承了spring中的扫描类的，只是覆写了一些方法
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
}
````
#### 在创建了这个类之后，可以看到最后会调用一个scan方法，同时传入了文件路径。
````
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
  	// 调用父类的doScan方法，得到对应的的BeanDefinition
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
    	// 在得到bean之后，调用自己实现的逻辑对其做二次加工
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
  
  // 二次加工扫描得到的BeanDefinition
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()
          + "' and '" + beanClassName + "' mapperInterface");

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      // 新增一个构造函数，并把当前bean的类名作为构造参数
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      // 将bean的类改为mapperFactoryBean类型
      definition.setBeanClass(this.mapperFactoryBean.getClass());
		
      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      // 新增一些bean definition的属性，但是一般为null
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        // 将自动注入的模式设置为根据type
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
````
#### 大致看了一下代码，可以发现整个加载的流程是会加载所有注解了@Mapper的类，然后将其转成mapperFactoryBean工厂类，而这个步骤的原因在后面我们也会慢慢找到原因，在本文中我们先看到这一步。然后看到我们真正使用的时候是怎么注入的。
````
 @Autowired
 private UserMapper userMapper;
````
#### 使用非常简单，所有问题都交由springboot完成，我们会直接通过刚才的工程类得到相应的Object。
````
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
````
#### 这块代码我们放在下一篇文章中去深究，总结一下我们刚看了@Mapper注解的类是怎么被spring框架加入到bean容器的。现在我们再简单看一下@MapperScan注解是怎么工作的。




.....TODO...



#### 上面我们看到了我们的mapper怎么被注册到spring容器中，现在看看怎么重新注入到我们的项目里。上面的文章我们也说到了我们所有的bean在注册的时候都会被改为MapperFactoryBean类型的工厂类，那么相应的，在bean的注入的时候，spring框架会通过工厂类的getObject方法得到我们想要的类。
````
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
````
#### 可以看到总共分为两个步骤，一个是获取SqlSession，也就是每次操作的会话，第二步是根据对应的mapper接口从这个会话中获取对应的Mapper。我们一步步断点代码会发现，最终会走到MapperRegistry的getMapper方法中，我们直接看到这段代码。
````
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  	// 通过type，得到mapperProxyFactory类
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
    	// 再通过mapperProxyFactory类，在当前会话中，得到一个新的实例
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
````
#### 所以继续看到mapperProxyFactory实例化的过程
````
  public T newInstance(SqlSession sqlSession) {
  	// 先得到一个MapperFactory对应
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
  
  // 又是熟悉的proxy模式，用mapperProxy对我们写的mapper类进行增强
  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
````
#### 至此我们就得到了一个由MapperProxy增强代理的mapper对象，并注入到了我们的项目中。我们就可以直接用这个类来完成我们想要的操作了。
