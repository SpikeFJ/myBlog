这篇主要关注下事件是如何通过管道一步步分发到worker线程进行处理, 而worker线程, 则主要用于处理客户端的读写事件

根据上一篇的分析，调用流程：

> logHandle-->ServerBootstrapAcceptor-->用户处理器


1. 连接接入

首先`NioEventLoop`作为一个持续监听用户连接的入口，在其`run`方法维护着一个死循环，

`run`方法同时处理IO事件和普通任务，这里我感觉代码不是很直观，虽然通过`ioRatio`对IO事件和普通任务做了一定的优先级侧重，

> 个人认为从源码角度看，职责不够清晰，如果没有性能考虑，应该拆分成两个线程

还是分析调用流程吧：

> `AbstractChannel.unsafe.read`()--->`doReadMessages`-->

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());

    try {
        if (ch != null) {
            //注意此NioSocketChannel的构建
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}
```
在`doReadMessages`中，通过调用`channel.accept`得到客户端`socketchannel`,
将`ServerSocketChannel`和客户端`socketChannel`包装成一个`NioSocketChannel`,存放到`List<Object>`中

我们需要注意下`NioSocketChannel`的构造函数，因为这里隐含了`workGroup`的绑定事件
```java
//NioSocketChannel.java
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}

//AbstractNioByteChannel.java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    //这里指明了客户端需要绑定的监听事件：OP_READ
    super(parent, ch, SelectionKey.OP_READ);
}
```

> ----> 循环调用 `pipeline.fireChannelRead`(`NioSocketChannel`)
>    ----->`HeadContext`--->`LogHandler`-->`ServerBootstrapAcceptor`

下面是ServerBootstrip.ServerBootstrapAcceptor.java

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    //msg就是NioSocketChannel，包含ServerSocketChannel和客户端socketChannel
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

后续的`register`流程和`bosswork`相同，不再赘述。


从`bossGroup`传递过来的`channel`对象，在`workgroup`处会注册其`read/write`事件，

即同一个`channel`贯穿 `bossGroup`和`workGroup`，

在`bossGroup`处注册的是`accept`事件，在`workGroup`处注册的是`read/write`事件