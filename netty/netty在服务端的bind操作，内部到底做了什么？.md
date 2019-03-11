# netty在服务端的bind操作，内部到底做了什么？

### 看看调用方式
```
	int PORT = 8887;
    ServerBootstrap b = new ServerBootstrap();
    // ... 此处省略一些ServerBootstrap绑定一些handle之类的操作
	// Start the server.
	ChannelFuture f = b.bind(PORT).sync();
```
### 继续跟进去AbstractBootstrap代码
```
    public ChannelFuture bind(int inetPort) {
    	// 只给定端口号的话，会用local:port的形式组成address
        return bind(new InetSocketAddress(inetPort));
    }
    
    
    public ChannelFuture bind(SocketAddress localAddress) {
    	// 检验一下一些需要的成员变量是否存在
        validate();
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);
    }
    
    
    // 接下来的这个bind才是真正的重点
private ChannelFuture doBind(final SocketAddress localAddress) {
		// 简单说起来就是这个方法会初始化一个channel,因为是异步处理，所以会直接先返回一个ChannelFuture
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        // 如果操作失败,直接返回
        if (regFuture.cause() != null) {
            return regFuture;
        }
		// 因为是异步，所以此时可能会有两种情况, 若此时已经成功，则继续下面的绑定步骤
        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            // 若没有，则开一个监听器，等待完成或者失败
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
```
