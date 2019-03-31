# bean在spring中的生命周期
#### 对于普通的Java对象，当new的时候创建对象，当它没有任何引用的时候被垃圾回收机制回收。而由Spring IoC容器托管的对象，它们的生命周期完全由容器控制。Spring中每个Bean的生命周期如下：



applicationContext使用的两个组件具体实现为
	PathMatchingResourcePatternResolver
	StandardEnvironment


在启动时，会进入refresh


方法主要包含以下几个步骤
```
	prepareRefresh();

	// Tell the subclass to refresh the internal bean factory.
	ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

	// Prepare the bean factory for use in this context.
	prepareBeanFactory(beanFactory);

	try {
		// Allows post-processing of the bean factory in context subclasses.
		postProcessBeanFactory(beanFactory);

		// Invoke factory processors registered as beans in the context.
		invokeBeanFactoryPostProcessors(beanFactory);

		// Register bean processors that intercept bean creation.
		registerBeanPostProcessors(beanFactory);

		// Initialize message source for this context.
		initMessageSource();

		// Initialize event multicaster for this context.
		initApplicationEventMulticaster();

		// Initialize other special beans in specific context subclasses.
		onRefresh();

		// Check for listener beans and register them.
		registerListeners();

		// Instantiate all remaining (non-lazy-init) singletons.
		finishBeanFactoryInitialization(beanFactory);

		// Last step: publish corresponding event.
		finishRefresh();
	}

	catch (BeansException ex) {
		// Destroy already created singletons to avoid dangling resources.
		destroyBeans();

		// Reset 'active' flag.
		cancelRefresh(ex);

		// Propagate exception to caller.
		throw ex;
	}
```