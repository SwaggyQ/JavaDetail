# Spring的依赖注入

## 前文
#### 本文详解一下关于Spring控制反转的具体实现，也就是怎么从beanFactory中直接得到我们想要的bean。
## 正文
####  直接看到最简单的一种对Spring的运用，
```
		// application.xml		
		<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="myTestStr" class="com.sugu.spring.bean.MyTestStr"></bean>
</beans>



		// Application.java
		ApplicationContext applicationContext = new ClassPathXmlApplicationContext("application.xml");
        MyTestStr myTestStr = (MyTestStr) applicationContext.getBean("myTestStr");
        System.out.println(myTestStr.getTestStr());
```
#### 上面的例子是最基本的控制反转的例子，今天本文只会关注一行代码，也就是
```
        MyTestStr myTestStr = (MyTestStr) factory.getBean("myTestStr");
```
#### 直接跟进去，看到AbstractBeanFactory类的实现
```
	// 我们需要传入的就是我们想要的类的name，非常简单
	// 再对后面的三个参数做一个简单的记录
	// requiredType,args,typeCheckOnly
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
	
	// 通过beanName得到对应的实现的具体实现
	protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
		// 先简单对bean的name做一个处理，里面的逻辑主要只有将FactoryBean的name前的&号去掉。
		// 也就是若传入的为&beanName，则处理得到为beanName
		// 因为用户可以指定需要得到的是工厂类，还是工厂类对应的实体类。所以用&前缀作为区分，即带有&符号，则代表想要得到工厂类
		final String beanName = transformedBeanName(name);
		// 这个就是我们之后会返回的对应的bean
		Object bean;
		...
	}
```
##### 方法一开始就只做了对bean名字的简单处理，去掉了&。接下来的部分，主要是为了满足bean加载的单例。所有的bean在同一个容器中只会被加载一次，只会会用到，都会直接从缓存中读取。并且为了解决循环依赖的问题，在bean还没创建好的时候，就会先暴露对应的ObjectFactory，这样下次创建对应的bean的时候，就可以直接拿到对应的objectFactory。关于循环依赖的这个事情，也会在后面讲到。不知不觉，已经挖了两个叫做工厂bean和循环依赖的坑了。
```
	// Eagerly check singleton cache for manually registered singletons.
	Object sharedInstance = getSingleton(beanName);
		
	public Object getSingleton(String beanName) {
		return getSingleton(beanName, true);
	}
	
	// 真正起作用的方法
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// 先检查在缓存中，对应的bean name是不是已经生成了对应的bean instance
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			// 如果还没生成，则需要先加锁，再创建对应的instance
			synchronized (this.singletonObjects) {
				// 是否先存在了正在新建的bean对象
				singletonObject = this.earlySingletonObjects.get(beanName);
				// 若没有正在加载的对象，以及允许先提前暴露引用对象(默认为true)
				if (singletonObject == null && allowEarlyReference) {
					// 检查是否已经暴露了对应的ObjectFactory
					ObjectFactory singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						// 若已暴露了ObjectFactory，则直接生成Object返回，并声明正在创建中，并删除ObjectFactory
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		// 若没得到instance，则返回null
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```
#### 如果是第一次去获得bean，则会返回null，并在后面的代码中去得到
```
	final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
```
#### 同样的，会先去得到bean的定义，然后根据bean配置的scope，来判断以什么方式去创建bean。默认为sigleton模式，即会调用到下面的方法中
```
	if (mbd.isSingleton()) {
		sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
			public Object getObject() throws BeansException {
				...			
			});
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
	}
```
#### 可以看到，主要是调用了getSingleton方法
```
	synchronized (this.singletonObjects) {
		// 同样先检查缓存中是否存在
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while the singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
			}
			// 创建前再确认一次
			beforeSingletonCreation(beanName);
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<Exception>();
			}
			try {
				// 调用匿名类的方法
				singletonObject = singletonFactory.getObject();
			}
			catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				// 再检查一遍
				afterSingletonCreation(beanName);
			}
			addSingleton(beanName, singletonObject);
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```
#### 上面调用了很多方法，但是除去很多的检查的方法，最重要的就是singletonFactory.getObject()这一句。这个就是之前定义的匿名类，所以我们回去看看这个方法做了什么操作
```
	public Object getObject() throws BeansException {
		try {
			return createBean(beanName, mbd, args);
		}
		catch (BeansException ex) {
			// Explicitly remove instance from singleton cache: It might have been put there
			// eagerly by the creation process, to allow for circular reference resolution.
			// Also remove any beans that received a temporary reference to the bean.
			destroySingleton(beanName);
			throw ex;
		}
	}
```
#### 就是调用了createBean方法，继续看进去

























































#### 上述代码主要是为了检查是否已经在缓存中存在了相应的bean instance，避免重复创建。现在再来看看接下来做了什么。
```		
		// 如果已经得到了共享的instance，且未传入参数
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		
		
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

		// 若传入的bean name由&开头，但是instance没有实现FactoryBean接口，则会报错
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		// 如果实例对象不是一个工厂类，或者想要直接得到工厂类
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}
		// 如果执行到了这一步，则代表想要得到工厂类对应的实体类
		Object object = null;
		if (mbd == null) {
			// 缓存查看是否已经缓存了对应的实体类，若有则直接返回
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// 将之前得到的instance转化为工厂类的对象，因为后面需要他来得到对应的bean对象
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// 若之前加载的配置中定义了当前的bean name
			if (mbd == null && containsBeanDefinition(beanName)) {
				// 得到当前bean相关的配置信息 
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			// mdb是否由程序直接生成?
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			// 从工厂类中得到对应的bean
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```
#### 上面的代码主要逻辑还集中在最后的getObjectFromFactoryBean方法。前面主要对bean name做了判断，并仅当需要返回工厂类的实体类的时候，才会执行到最后的方法。现在再看看里面做了什么。
```
	protected Object getObjectFromFactoryBean(FactoryBean factory, String beanName, boolean shouldPostProcess) {
		// 判断是否需要单例，如果是，且已经存在了bean instance
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				// 检查缓存中是否已经存在工厂类的实体类，若有则直接返回
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					// 如果还是找到，则需要进行创建
					object = doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess);
					// 创建成功后，需要在缓存中保留一份
					this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
				}
				return (object != NULL_OBJECT ? object : null);
			}
		}
		else {
			// 若不是单例模式，或者还没有bean instance存在，则直接创建
			return doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess);
		}
	}
```
#### 还是没看到我们想看的核心代码，只是对单例模式，做了一些简单的控制。核心代码还是得接下去看
```
	private Object doGetObjectFromFactoryBean(
			final FactoryBean factory, final String beanName, final boolean shouldPostProcess)
			throws BeanCreationException {

		Object object;
		try {
			// 是否需要权限控制
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						public Object run() throws Exception {
								return factory.getObject();
							}
						}, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				// 核心代码只有这一句
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// 如果创建结果为null，或者此时正有其他操作正在创建，则抛出异常
		if (object == null && isSingletonCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(
					beanName, "FactoryBean which is currently in creation returned null from getObject");
		}
		// 若创建成功，且需要调用Object的后处理器(即之前保留的是否为用户自创建)
		if (object != null && shouldPostProcess) {
			try {
				object = postProcessObjectFromFactoryBean(object, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(beanName, "Post-processing of the FactoryBean's object failed", ex);
			}
		}

		return object;
	}
```
#### 核心代码很简单，就是调用工厂类的getBean方法，注意最后可能还需要处理一个后处理器。
// 暂时也先留个坑
```
	@Override
	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
		return applyBeanPostProcessorsAfterInitialization(object, beanName);
	}
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			result = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
		return result;
	}
```

#### 上面都是当从单例的缓存中得到结果的做法。如果此时未得到结果，则需要执行不同的流程。回到doGetBean方法中
```
		// 如果正在有其他应用线程正在创建同样的bean，则跑错
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// 检查bean对应的配置是否已经存在
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			// 若是，则直接创建
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}
		
		// 标记当前bean已创建
		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
		checkMergedBeanDefinition(mbd, beanName, args);
```

#### 先简单过一遍上面的代码，最重要的就是得到了mbd，也就是merged bean define。也就是我们配置定义的相应的bean的相关属性。以下是我这个最简单的bean的配置对应的mbd。


Root bean: class [com.sugu.spring.bean.MyTestStr]; scope=singleton; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in class path resource [application.xml]

#### 也就是最重要的是否为抽象类，是否为单例类似的属性。得到mdb之后，就会正式进步createBean的过程了。
```
	// bean默认为单例实现
	if (mbd.isSingleton()) {
		sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
			// 实现了抽象类的getObject方法，在后面会用到
			public Object getObject() throws BeansException {
				try {
					return createBean(beanName, mbd, args);
				}
				catch (BeansException ex) {
					// Explicitly remove instance from singleton cache: It might have been put there
					// eagerly by the creation process, to allow for circular reference resolution.
					// Also remove any beans that received a temporary reference to the bean.
					destroySingleton(beanName);
					throw ex;
				}
			}
		});
		// 之前提过，具体就是判断是否为工厂类
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
	}
```
#### 重点看看这个getSingleton方法
```
public Object getSingleton(String beanName, ObjectFactory singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			// 还没有缓存对象，所以需要现在创建
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while the singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				// 创建前的准备，主要还是判断这个bean当前是不是正在被创建
				beforeSingletonCreation(beanName);
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<Exception>();
				}
				try {
					// 调用上述的实现的方法来创建对用的bean
					singletonObject = singletonFactory.getObject();
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					// 创建后的操作，主要判断创建完成
					afterSingletonCreation(beanName);
				}
				// 更新单例缓存
				// 具体为
				// this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
				// this.singletonFactories.remove(beanName);
				// this.earlySingletonObjects.remove(beanName);
				// this.registeredSingletons.add(beanName);
				addSingleton(beanName, singletonObject);
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}
```
#### 以上简单来说，其实就是调用了createBean方法
```
	@Override
	protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		// 提前确认class是可以加载的
		resolveBeanClass(mbd, beanName);

		// Prepare method overrides.
		try {
			// 验证并准备覆盖的方法
			mbd.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbd);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}
		
		Object beanInstance = doCreateBean(beanName, mbd, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
```
#### 实在太绕了












Spring Bean 支持 5 种 Scope ，分别如下：

Singleton - 每个 Spring IoC 容器仅有一个单 Bean 实例。默认
Prototype - 每次请求都会产生一个新的实例。
Request - 每一次 HTTP 请求都会产生一个新的 Bean 实例，并且该 Bean 仅在当前 HTTP 请求内有效。
Session - 每一个的 Session 都会产生一个新的 Bean 实例，同时该 Bean 仅在当前 HTTP Session 内有效。
Application - 每一个 Web Application 都会产生一个新的 Bean ，同时该 Bean 仅在当前 Web Application 内有效。





SingletonObjects : bean name -> bean instance
SingletonFactories : bean name -> Object Factory
earlySingletonObjects : bean name -> bean instance ,当bean还在创建的时候，可以从这边得到，所以解决循环依赖的问题
reigsteredSingleton : 保存当前已注册的bean