本篇继续分析`netty`的启动流程

我们从`bootstrap.bind(30000)`入手,下面是代码流程：
```java
public ChannelFuture bind(SocketAddress localAddress) {
    //1.首先进行参数校验
    validate();
    //2. 执行具体绑定
    return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
}

public B validate() {
    if (group == null) {
        throw new IllegalStateException("group not set");
    }
    if (channelFactory == null) {
        throw new IllegalStateException("channel or channelFactory not set");
    }
    return self();
}


private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

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
最后会调用`AbstractBootStrip`的`doBind()`,`doBind`主要做了以下两件事：

1. `初始化并注册通道--initAndRegister`
2.  绑定--`doBind0`

我们这里先分析初始化和注册流程，下一篇分析如何绑定

```java
final ChannelFuture initAndRegister() {
    //1.通道初始化
    Channel channel = null;
    try {
        //1.反射生成 NioServerSocketChannel，对应启动类中.channel(NioServerSocketChannel.class)
        //此处要注意  super(null, channel, SelectionKey.OP_ACCEPT);这行代码指明了serverChannel关注的操作
        channel = channelFactory.newChannel();
        //2.初始化通道
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            channel.unsafe().closeForcibly();
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }

    //2.通道注册
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    return regFuture;
}
```

# 一. 初始化通道

```java
void init(Channel channel) {
    //1.设置Tcp相关参数
    setChannelOptions(channel, newOptionsArray(), logger);
    //2.设置netty相关参数
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

    //3.调用`pipeline`的`addLast`添加`ChannelInitializer`,在`ChannelInitializer`的`initChannel`中会添加`ServerBootstrapAcceptor`对象。
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                     //ServerBootstrapAcceptor负责处理连接接入，后续会转交给workGroup处理，所以这里将currentChildGroup，currentChildHandler等对象传递给ServerBootstrapAcceptor负责处理连接接入
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

这里要注意`pipeline`第一次调用`addLast`时有很多初次调用逻辑，我们跟踪到`DefaultChannelPipeline`

```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        //将handle对象包装成一个DefaultChannelHandlerContext对象，DefaultChannelHandlerContext既持有pipeline又包含handle，所以说它是两者的通讯桥梁
        newCtx = newContext(group, filterName(name, handler), handler);

        //加入到链表中
        addLast0(newCtx);

        //如果通道尚未注册，则添加一个延迟add任务
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        //如果已注册且执行器且当前线程和loop线程是同一个线程，则传递并调用handleAdd事件
        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            callHandlerAddedInEventLoop(newCtx, executor);
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}

//添加一个延迟调用任务，实际调用时要等到channel注册后，
private void callHandlerCallbackLater(AbstractChannelHandlerContext ctx, boolean added) {
        assert !registered;
		//根据上述传递的true，得知生成了一个PendingHandlerAddedTask对象
        PendingHandlerCallback task = added ? new PendingHandlerAddedTask(ctx) : new PendingHandlerRemovedTask(ctx);
    	//pendingHandlerCallbackHead是链表头，只需要找到pendingHandlerCallbackHead就能执行后续所有的延迟调用
        PendingHandlerCallback pending = pendingHandlerCallbackHead;
        if (pending == null) {
            //第一次初始化pendingHandlerCallbackHead
            pendingHandlerCallbackHead = task;
        } else {
			//如果不是首次添加，则添加到pendingHandlerCallbackHead的尾结点
            while (pending.next != null) {
                pending = pending.next;
            }
            pending.next = task;
        }
    }
```

`PendingHandlerAddedTask`的执行逻辑只能在通道注册后才能执行，所以其继承`runnable`，

本身维护了两个属性：1.`channelContext` 2.`next`

其`run`方法主要就是执行`callHandlerAdded0`，而后续又是利用`channelContext.callHandlerAdded`

最终还是调用了 `channelContext`对应的`handle.handlerAdded`

# 二. 注册通道

上述的初始化暂告一段落，回到通道注册上

```java
 //AbstractBootStrip.cs
 ChannelFuture regFuture = config().group().register(channel);
    |
 //SingleThreadEventLoop.cs
  promise.channel().unsafe().register(this, promise);
```

此处调用`group`的`register`，而`group`会委托内部`next.register`,最终，实际执行在`SingleThreadEventLoop.register`中

可以看到后续调用翻转了，主体对象变成了`channel.register`,`channel`使用内部类`unsafe`执行注冊。

`AbstractChannel.AbstractUnsafe.register`如下：

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(eventLoop, "eventLoop");
    //1.是否已注册
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    //2.是否是NioEventLoop
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

   //此处维护了channel和loop的关系，即channel包含loop引用
    AbstractChannel.this.eventLoop = eventLoop;
    //3.下面的代码后面会经常遇到
    //主要流程是：如果在同一线程中，则直接执行，否则通过线程执行；是为了保证I/O事件以及用户定义的I/O事件处理逻辑在一个线程中处理
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```

上述注册流程如下：

1. 判断是否已注册、是否是`NioEventLoop`,是则记录异常直接返回

2. 执行：如果在同一线程中，则直接执行，否则通过线程执行

   **注意：这里是`eventLoop`第一次调用`execute`，会在内部启动轮询线程**

####  `register0`

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        //此处完整真正的通道注册
        doRegister();
        neverRegistered = false;
        registered = true;
        //触发handleadd事件，加入用户自定义的handle
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        //触发handleRegister事件
        pipeline.fireChannelRegistered();

        if (isActive()) {
            if (firstRegistration) {
                //触发avtive事件
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
protected void doRegister() throws Exception {
	//0表示仅仅是注册，不关注任何事件
   selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);

 }
```

由上述代码可以看出注册主要分为以下几个步骤：

1. 通过`channel.doRegister`完整实际的注册工作
2. 通过`invokeHandlerAddedIfNeeded`调用`handler.add`将处理业务逻辑的用户`Handler`添加到管道
3. 触发`register`事件
4. 触发`active`事件