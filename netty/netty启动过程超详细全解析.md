# netty启动过程超详细全解析
## 前言
#### 反反复复的看netty好几次了，一直徘徊在入门阶段，最近准备再冲击一下，正好发现看出了一点心得，所以准备在这边记录一下。本文会长文断点级别的梳理netty的启动流程，会尽量把流程梳理清楚，所以文章会比较繁琐比较枯燥，现在开始吧。
## 正文
#### 废话不多说，我们直接从netty的一个服务端启动的demo入手。
````
/**
 * demo是我之前看闪电侠的netty专栏复制下来的，感兴趣自行搜索闪电侠netty专栏，业界大佬
 * @author 闪电侠
 */
public class NettyServer {
    public static void main(String[] args) throws InterruptedException {
        ServerBootstrap serverBootstrap = new ServerBootstrap();

        NioEventLoopGroup boos = new NioEventLoopGroup();
        NioEventLoopGroup worker = new NioEventLoopGroup();
        serverBootstrap
                .group(boos, worker)
                .channel(NioServerSocketChannel.class)
                .handler(new SimpleServerHandler())
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    protected void initChannel(NioSocketChannel ch) {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new SimpleChannelInboundHandler<String>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext ctx, String msg) {
                                System.out.println(msg);
                            }
                        });
                    }
                })
                .bind(8000).sync();
    }

    private static class SimpleServerHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelActive");
        }

        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelRegistered");
        }

        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
            System.out.println("handlerAdded");
        }
    }
}
````

#### 如果我们在自己的ide中运行以上demo，我们就会在8000端口开启了一个netty的服务端了，非常方便。可以看到，开头三行分别创建了3个对象，其中两个NioEventLoopGroup对象，之后会注册到ServerBootstrap对象上