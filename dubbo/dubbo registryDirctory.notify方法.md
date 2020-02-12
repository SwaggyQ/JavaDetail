dubbo registryDirctory.notify方法

```java
this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
```

通过从注册中心的返回，刷新内部的invokers