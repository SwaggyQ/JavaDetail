dubbo 扯扯淡  appendProperties方法

set注入参数

从 System 中获取属性key值的优先通讯是：dubbo.provider.+ 当前类 Id + 当前属性名称 >dubbo.provider.+ 当前属性名称 为 key 获取值.

首先从特定properties文件加载属性：首先System.getProperty("dubbo.properties.file")获取到文件路径，如果获取不到就会试图加载 dubbo 的默认的路径dubbo.properties加载。获取属性的 key 和上面从 System 里面获取的规则一样。



