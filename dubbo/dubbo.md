dubbo

```
registry.register(
```



```
registry.subscribe(url, this);
```







```` registry.notify(url, this);
registry.notify()
````



````  registryDirectory.notify()
registryDirectory.notify()
````





registryDirectory作为invokers的容器，通过cluster.join()方法包装成一个invoker对外暴露，同时registryDirectory又作为一个notifyListener，收到registry的订阅回调。