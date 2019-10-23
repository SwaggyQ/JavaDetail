

# dubbo的Directory和Cluster, Router 的作用以及关系?

## 前言

本文将从一个细节出发，以小见大窥探一下关于dubbo服务的引入过程。为了说清楚这三者的关系，本文会稍微讲一下服务引用的过程。

## 正文

熟悉源码的人应该不难发现，我们服务导入的关键起始类是ReferenceBean，这个是一个工厂bean的实现，同时继承了ReferenceConfig，由父类实现大部分的导入逻辑。由于是一个工厂类，所以很容易发现在我们代码中真正注入对应的dubbo服务时，会调用FactoryBean的getObject()方法，实例化一个真正的对象。而这个方法也正是我们今天想讲的逻辑入口，我们看到对应的方法。



````java
    // ReferenceBean.java
    @Override
    public Object getObject() throws Exception {
        return get();
    }
    
    // ReferenceConfig.java
    public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("Already destroyed!");
        }
        if (ref == null) {
            init();
        }
        return ref;
    }
````

所以最后返回的这个ref，就应该是我们真正想要调用的dubbo服务的一个代理类，所以最重要的就是这个ref是怎么被生成的，所以继续看到init()方法。

````java
    private void init() {
        ......
        ref = createProxy(map);
        ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
        ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
    }
````

整个方法涉及非常多的参数提前校验以及初始化的细节，由于不是本文的重点，所以先直接略过了，直接看到这个createProxy方法，因为我们最关注的的ref是由这个方法初始化的。

````java
    private T createProxy(Map<String, String> map) {
        if (isJvmRefer) {
            invoker = refprotocol.refer(interfaceClass, url);
        } else {
            if (urls.size() == 1) {
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // use last registry url
                    }
                }
                if (registryURL != null) { // registry url is available
                    // use AvailableCluster only when register's cluster is available
                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                } else { // not a registry url
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        }
        // create service proxy
        return (T) proxyFactory.getProxy(invoker);
    }
````

为了更清楚的看到整个方法的脉络，我省略了很多代码。由上面的代码框架可以看出整个流程，第一步都是通过refprotocol.refer()方法，得到invoker，然后若得到的是多个invoker，则需要用cluster.join()方法，对其做一个附加操作。最后通过proxyFactory.getProxy()得到对应的invoker的代理类。所以重要的三个方法就是

````java
1. refprotocol.refer(interfaceClass, url);
2. cluster.join(new StaticDirectory(u, invokers));
3. proxyFactory.getProxy(invoker);
````

这也正好引出了我标题中的两个重要的接口，Directory和Cluster，还剩一个Router接口会在我们后面的讲解中出场。

现在我们先看到第一个方法。

###Protocol.refer()



### Cluster.join()

###  ProxyFactory.getProxy()



