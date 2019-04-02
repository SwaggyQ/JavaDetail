# spring aop 
    <aop:aspectj-autoproxy/>
	自定义的标签由，AopNamespaceHandler类解析
	最后由AspectJAutoProxyBeanDefinitionParser 解析
	所有BeanDefinitionParser的所有子类，都由parse方法进入
	
## registerAspectJAnnotationAutoProxyCreatorIfNecessary
	注册或升级AnnotationAwareAspectJAutoProxyCreator
	registerAspectJAnnotationAutoProxyCreatorIfNecessary
		如果已存在自动代理创建器，则需要考虑优先级
		
## useClassProxyingIfNecessary
	对于proxy-target-class属性的处理
		默认为false，默认使用jdk的动态代理，可以设置为true，则代表强制使用cglib
		<aop:aspectj-autoproxy proxy-target-class="true"/>
		
		使用cglib会有两个问题
			无法advise Final方法，因为他们不能会覆写
			需要将CGLIB二进制发行包放在classpath下
		
		
		cglib和jdk动态代理的细节差别
		jdk动态代理： 其代理对象必须是某个接口的实现，他是通过在运行期间创建一个接口的实现类来完成对目标对象的代理
		cglib代理： 实现原理类似JDK动态代理，只是他在运行期间生成的代理对象是针对目标类扩展的子类。CGLIB是高效的代码生承包，底层依靠ASM操作字节码实现。性能在1.6之前比jdk动态代理效率强。之后jdk效率好。
	对于expose-proxy属性的处理
		没看懂
		
		
## 再看到AnnotationAwareAspectJAutoProxyCreator类中，实现了BeanPostProcessor接口
	AbstractAutoProxyCreator的postProcessBeforeInstantiation方法，过滤掉之后
	getAdvicesAndAdvisorsForBean方法创建代理
	createProxy 创建代理
	创建代理包括两步
		1：获取增强方法或者增强器
		2：根据获取的增强进行代理
		

## findEligibleAdvisors方法
	findCandidateAdvisors
	findAdvisorsThatCanApply
	
	1：获取所有在beanFactory中注册的Bean都会被提取
	2：遍历所有beanNanme，并找出所有AspectJ注解
	3：对提取的类进行增强器的提取
	4：将提取结果加入缓存
	
	
## 得到所有Advisor之后，开始创建代理 createProxy
	新建一个ProxyFactory，然后加入所有的advisor
	
	然后再createAopProxy().getProxy(classLoader);
	
	在DefaultAopProxyFactory类中，createAopProxy方法选定jdk动态还是cglib
	
	
	
	
	
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface()) {
				return new JdkDynamicAopProxy(config);
			}
			if (!cglibAvailable) {
				throw new AopConfigException(
						"Cannot proxy target class because CGLIB2 is not available. " +
						"Add CGLIB to the class path or specify proxy interfaces.");
			}
			return CglibProxyFactory.createCglibProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
		
		
		
		如果目标对象实现接口，默认采用jdk动态代理，除非强制使用cglib
		如果没有实现接口，必须采用cglib，spring对自动转换
		
		
		jdk动态代理只能对实现了接口的类生成代理，不能针对类
		cglib是针对类实现代理，对指定的类生成一个子类，覆盖方法，所以类或者方法最好不要是final
		
		
		
		jdk动态代理最重要的是InvocationHandler的创建
		重写构造函数，传入代理对象
		icnoke方法，实现增强逻辑
		getProxy方法，内部就是传入classLoader，interfaces，this
		
		
		
		spring aop中jdk动态代理的类是JdkDynamicAopProxy
		cglib的类是CglibProxyFactory.createCglibProxy(config);