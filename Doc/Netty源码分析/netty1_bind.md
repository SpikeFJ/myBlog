上一篇提到了`channel`只是注册了，但并没有监听任何事件，

本篇主要分析下`channel`是如何绑定监听事件的

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    //绑定
    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();

                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

继续跟进

```java
//AbstractChannel.java
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}
//DefaultChannelPipeline.java
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return tail.bind(localAddress, promise);
}
//AbstractChannelHandlerContext.java
public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(localAddress, "localAddress");
    if (isNotValidPromise(promise, false)) {
        // cancelled
        return promise;
    }
	//这里next指向了pipeline的第一个context对象
    final AbstractChannelHandlerContext next = findContextOutbound(MASK_BIND);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeBind(localAddress, promise);
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeBind(localAddress, promise);
            }
        }, promise, null, false);
    }
    return promise;
}

```

这里调用流程需要注意：

`DefaultChannelPipeline.bind`--->`tail.bind`--->`next.invokeBind`

`invokeBind`实际是调用了`handler`的`bind`方法

```java
private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
    if (invokeHandler()) {
        try {
            ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    } else {
        bind(localAddress, promise);
    }
}
```

![](D:\gitResource\myBlog\Resource\netty\netty2_2020-06-06_17-32-27.png)

观察下`loggingHandler`,

```java
public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
    if (logger.isEnabled(internalLevel)) {
        logger.log(internalLevel, format(ctx, "BIND", localAddress));
    }
    //此处是重点，将bind委托给下一个ctx处理，下一个就是headContext
    ctx.bind(localAddress, promise);
}
```

继续观察`headContext`的`bind`方法

```java
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
    unsafe.bind(localAddress, promise);
    //这里就是终点，不会继续委托下一个ctx
}
```

**可以看到实际处理还是`AbstractChannel.unsafe`的`bind`**

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();

    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

	//...
    boolean wasActive = isActive();
    try {
        //绑定
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    //判断是否需要发起channelActive事件调用
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}
```



下面是`DefaultChannelPipeline`的`fireChannelActive`调用流程：

```java
//DefaultChannelPipeline.java
public final ChannelPipeline fireChannelActive() {
    //从头节点开始传递
    AbstractChannelHandlerContext.invokeChannelActive(head);
    return this;
}

//AbstractChannelHandlerContext.java
static void invokeChannelActive(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelActive();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelActive();
                }
            });
        }
}
//调用context中对应的handle.ChannelActive
 private void invokeChannelActive() {
        if (invokeHandler()) {
            try {
                //默认的hadler()实现指向this
                ((ChannelInboundHandler) handler()).channelActive(this);
            } catch (Throwable t) {
                invokeExceptionCaught(t);
            }
        } else {
            fireChannelActive();
        }
}

 public void channelActive(ChannelHandlerContext ctx) {
     ctx.fireChannelActive();

     readIfIsAutoRead();
 }

 private void readIfIsAutoRead() {
     if (channel.config().isAutoRead()) {
         channel.read();
     }
 }
public Channel read() {
    pipeline.read();
    return this;
}
public final ChannelPipeline read() {
    tail.read();
    return this;
}

public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}


```

**继续跟踪到`AbstractUnsafe`的`beginRead`()方法中:**

```java
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    //interestOps=0，所以进入if块中
    if ((interestOps & readInterestOp) == 0) {
        //readInterestOp是NioServerSocketChannel对象新建时赋值的。
        //readInterestOp = SelectionKey.OP_ACCEPT
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
//NioServerSocketChannel通过工厂反射时，默认监听SelectionKey.OP_ACCEPT
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}

```

