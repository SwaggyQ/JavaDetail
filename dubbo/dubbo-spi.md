# dubbo -- SPI
## 前言
#### 今天开始好好记录一下学习dubbo的各种小细节。
## 正文
#### 一开始我们直接先通过dubbo官方的一个图来引入我们今天的主题![dubbo](img/dubbo-framework.jpg)。好的，相信如果大家是第一次看到这个图，肯定会被看的眼花缭乱。今天我们不会直接开始分析每个流程，直接先看到里面从上到下，有Service，Config，Proxy，Registry，Cluster，Monitor，Protocol，Exchange，Transport，Serialize，总共十层，每一层都是dubbo中非常重要的一个模块。在之后的文章中，我们就会对其进行分析。但是今天，我们先看到另一个事情。官方有这么一句话
> Service 和 Config 层为 API，其它各层均为 SPI 

#### 这句话我们要怎么去理解呢。其它层是SPI是什么意思呢？要解释这个，我们要先看到SPI是什么？其实SPI是java自带的一种机制，而Dubbo对其进行了更好的改造和适配，让其遍布于Dubbo源码的每个角落，每个SPI都可以看做一个扩展点。所以其中8层都是SPI，就是说明其他八层都可以供开发人员扩展自己需要的组件。
#### 我们直接在Dubbo中找到一个SPI相关的语句，直接看看到底是做了什么，以及为什么可以这么做。
```
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
#### 这是在ServiceConfig类中的一句，这个类就是我们所有关于dubbo的spring配置类最终的实体类。可以看到，这边声明了一个Protocol的变量，但是是用一种去load的过程。我们先去看看getExtensionLoad(Protocol.class)这个方法到底做了什么。
#### 在看得到具体的ExtensionLoad的时候，我们先插播一下这个类中的一些重要的变量，这几个变量是类的实现中比较容易混淆的,也是看懂这个类比较重要的点。

```
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
	
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();

    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();
	
	
	// 缓存已加载的扩展类
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();

    private final Map<String, Object> cachedActivates = new ConcurrentHashMap<String, Object>();
    
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
    
    private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    
    private volatile Class<?> cachedAdaptiveClass = null;
    
    private String cachedDefaultName;
    
    private volatile Throwable createAdaptiveInstanceError;

    private Set<Class<?>> cachedWrapperClasses;

    private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<String, IllegalStateException>();
```
#### 好了，先了解了上面的变量后，回到正文。
```
	// Protocol.java
	// 首先看一下这个Protocol类，是一个注解了SPI的接口，会有很多实现子类，所以可以猜测到这个方法就是要load出满足条件的他的子类
	@SPI("dubbo")
	public interface Protocol {
	}
	
	// ExtensionLoader.java
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    	// 校验一下传入的class是不是一个非空的注解了SPI的接口
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }
		 // 先得到是否已经缓存了相应的class的扩展类加载器，若是，则直接返回
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
        	// 如果不存在，也需要新建一个
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
    
    
    // 新建对应的class的扩展加载器
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        // 如果传入的不是extension工厂类，则和之前一样的逻辑得到扩展罗加载器，然后调用了getAdaptiveExtension方法，流程就是 
        // Protocol-start --> ExtensionFactory-start -->  ExtensionFactory-end --> Protocol-end
        // 假设这边已经完成了ExtensionFactory的过程
 		 // 所以回到之前的方法的话，EXTENSION_LOADERS中将会有两个type的扩展类加载器
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```
#### 以上的方法内部的逻辑也是清楚的，现在我们已经拿到了Protocol类的ExtensionLoader，接下来我们会看到下一步的getAdaptiveExtension方法。开始之前我们可以看一下ExtensionLoader类内部的方法，除了这个方法外，类似的方法还有一个叫做getExtension(String name)的方法。这两个方法就是，传入了name的话，就直接去加载对应的子类，如果是之前的方法，则会加载最适合的子类。显然直接加载指定的子类，这种情况是比较简单的，那么我们可以也从这种简单的方法入手。
```
	// 假设现在传入的name是dubbo，代表我们想加载DubboProtocol这个子类
    public T getExtension(String name) {
        if (name == null || name.length() == 0) {
            throw new IllegalArgumentException("Extension name == null");
        }
        // 默认的一种实现方式
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        // 这边的Holder代表装载一种扩展类的容器，也是和name一一对应
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        // 如果对应的扩展类还没有加载，则要初始化
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                		// 用双重检验的单例模式
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
    
    // 得到指定的扩展类
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        ...
    }
    
    private Map<String, Class<?>> getExtensionClasses() {
        // 得到所有已加载的扩展类，若还未加载则需要进行加载
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```
#### 这边先停一下，调用了好几次方法，接下来会调用到loadExtensionClasses方法，这个方法会从代码配置文件中，加载我们预先设定好的所有的扩展类，并进行缓存
```
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
		// 这边会得到在SPI注解中，value的值，默认为空，如果有些，则缓存一下default名字
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) {
                    cachedDefaultName = names[0];
                }
            }
        }
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        // 接下来会从项目中的几个配置文件中，根据类的全限名称得到对应的class类
        // 文件路径包括
        // META-INF/dubbo/
        // META-INF/dubbo/internal
        // META-INF/services/
        // 具体的加载过程，这边先忽略一下
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        return extensionClasses;
    }
```
#### 下面是我从dubbo的源码中截取的一个关于Protocol的配置
>文件名: org.apache.dubbo.rpc.Protocol
>
>filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=org.apache.dubbo.rpc.support.MockProtocol

#### 好了，不知道大家有没有被绕晕。从配置文件中加载所有的class类之后，会继续createExtension方法，所以我们再继续看这个方法。
```
private T createExtension(String name) {
        // 之前的操作让我们得到了name对应的class
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
        	// 是否缓存过class对应的实例
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            // 如果没有，则直接通过反射机制，得到对应的实例对象
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            ......
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```
#### 这时候我们已经通过name，得到了实例化的对象。好像已经满足了我们一开始的需求。但是在dubbo源码中接下来还进行了两步操作，也就是我上面......的地方，这也是dubbo官方说SPI实现了IOC和AOP的地方，我们现在先看看IOC的部分。
```
		// 调用了此方法
		 injectExtension(instance);
		 
private T injectExtension(T instance) {
        try {
        	// 回想一下，这个工厂类就是我们之前初始化过的ExtensionFactory
            if (objectFactory != null) {
            	// 遍历这个类的所有方法，针对所有只有setXXX的方法，且参数只有一个变量的方法进行依赖注入
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        Class<?> pt = method.getParameterTypes()[0];
                        if (ReflectUtils.isPrimitives(pt)) {
                            continue;
                        }
                        // 上面的两个条件，如果不能注入的，或者参数是基本类型的，则直接跳过，只对对象进行注入
                        // 
                        try {
                        	// 得到参数名称，例如 setObject1(Object1 object1)方法
                        	// 则得到property就是object1，也就是在类中依赖注入的变量的名称
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            // 通过工厂类，得到对应的类。这边ExtensionFactory有自己的几个实现
                            // Adaptive
                            // spi
                            // spring
                            // 具体也先不说了，spring通过配置文件，其他两个也都是通过ExtensionLoader方式的变种
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```
#### 上面的方法就是IOC的实现，将生成的实例化对象的对象变量都通过具体的实现注入。再来看下面的操作
```
		// 提一下cachedWrapperClasses，是在之前从各个配置文件加载对应的class的时候，会将构造器包含指定type的类视作装饰器，因为各个类之间都是装饰器模式来连通的，例如
		// public ProtocolFilterWrapper(Protocol protocol) { }
		// ProtocolFilterWrapper就会被记录在cachedWrapperClasses中，作为Protocol的一个Wrapper
		Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
            	// 遍历所有的包装类，实例化以及依赖注入
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
```
#### 以上就是我们通过一个name得到一个指定的扩展类的过程。我们之前也说了，这只是一种最简单的方法，不太会用到，比较常用到的还是getAdaptiveExtension方法
```
	public T getAdaptiveExtension() {
		// 看看是否缓存的最适合的扩展类实现
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
        	// 如果没有缓存，则需要新创建
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                        	// 得到最佳的扩展类是实现
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                        	// 如果加载失败了，则需要记录错误，之后抛出
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
```
#### 可以看到，上面的方法内部除了一些校验之类的代码外，真正的逻辑是createAdaptiveExtension方法。
```
    private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
    
    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
    
    private Class<?> createAdaptiveExtensionClass() {
        String code = createAdaptiveExtensionClassCode();
        ClassLoader classLoader = findClassLoader();
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```
#### 上面的调用链路还是清晰的，中间调用了一个getExtensionClasses方法，跟进去可以看到，在loadClass方法中，如果这个类标记了@Adaptive注解，则会存为cachedAdaptiveClass，那么在判断中会直接跳出，作为选择的最佳的扩展类。但是如果所有扩展类都没有加这个注解的话，就会开始骚操作了。继续看到最后的方法中的createAdaptiveExtensionClassCode，接下来的代码会比较神奇了哈哈。不要怕，继续看进去
```

```