

# 一. 基本样例

首先我们以一个常用的列子入手：

```java
//1.创建eventLoopGroup
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
        //2.指定channel对象
        .channel(NioServerSocketChannel.class)
        //3.bossGroup处理器
        .handler(new LoggingHandler(LogLevel.INFO))
        //4.workGroup处理器
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ChannelPipeline p = ch.pipeline();
                if (sslCtx != null) {
                    p.addLast(sslCtx.newHandler(ch.alloc()));
                }
                p.addLast(new DiscardServerHandler());
            }
        });
    //5.开始监听
    ChannelFuture f = b.bind(PORT).sync();
    f.channel().closeFuture().sync();
} finally {
    workerGroup.shutdownGracefully();
    bossGroup.shutdownGracefully();
}


```

**注意：后续代码分析只是列举出比较重要的部分，次要部分忽略。**

# 二. `NioEventLoopGroup`

首先我们关注`EventLoopGroup`对象的创建，其中`boss`负责连接的接入 `work`负责IO读写
```java
EventLoopGroup boss = new NioEventLoopGroup();
EventLoopGroup work = new NioEventLoopGroup();
```
从命名可以看出来`LoopGroup`应该包含/管理了一批`Loop`对象，`loop`代表无限循环，其对象内部维护着一个无限循环的结构。

继续回到代码：
> NioEventLoopGroup-->MultithreadEventLoopGroup-->MultithreadEventExecutorGroup-->AbstractEventExecutorGroup-->EventExecutorGroup

## 1. `NioEventLoopGroup`

`NioEventLoopGroup`类本身没有多少逻辑，主要就是提供了多个构造函数重载，最终还是调用了父类的构造函数。

在其父类`MultithreadEventLoopGroup`静态初始化块中，设定了默认线程数为处理器2倍的数目。

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
         //....
        //1. 创建线程执行器
        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        //2. 初始化loop对象
        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
              //...
            } finally {
                //3. 如果失败，则将之前成功启动的loop对象依次关闭
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }
                    //因为关闭使用的是异步优雅关闭，只是通知线程池关闭。有可能并未实际执行，需要检测其状态
                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }
        
        //创建一个选取器(这里和Nio的选择器区分开来)
        chooser = chooserFactory.newChooser(children);

        //对每一个loop对象添加终止事件
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        //将所有loop对象copy一个只读集合中
        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```
**总结下构造函数主要做了以下工作：**

1. **创建一个线程执行器ThreadPerTaskExecutor，该执行器的作用在于：后续启动NioEventLoopGroup组中的Loop对象时，不能串行执行，所以会利用该执行器开多个线程同时启动**
2. **通过`newChild`方法初始化`loop`对象，该方法由`NioEventLoopGroup`对象实现**
3. **创建一个选取器，该选取器的作用在一个`NioEventLoopGroup`维护着多个`NioEventLoop`对象，当连接到达时，应该使用哪一个`NioEventLoop`对象来处理？这就是选取器：`EventExecutorChooser`要负责的。**
4. **给每一个`loop`对象添加终止事件**
5. **将所有`loop`对象添加到一个只读数组中**



此处简单聊下选择工厂`EventExecutorChooserFactory`，该工厂类主要是提供一个`EventExecutorChooser`，该对象则是`eventLoopGroup`用来选择`eventLoop`的，默认工厂类提供了两种选择策略：

```java
//DefaultEventExecutorChooserFactory.cs
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}

//此处通过移位运算判断是否2的N次幂
private static boolean isPowerOfTwo(int val) {
    return (val & -val) == val;
}

//PowerOfTwoEventExecutorChooser.cs
public EventExecutor next() {
    return executors[idx.getAndIncrement() & executors.length - 1];
}

//GenericEventExecutorChooser.cs
public EventExecutor next() {
    return executors[Math.abs(idx.getAndIncrement() % executors.length)];
}

```

上述两种选择策略，功能都是一样的：依次选择下一个`eventLoop`对象对外提供服务，所以基本逻辑都是每次都是idx自增后对线程池长度取模，只是`PowerOfTwoEventExecutorChooser`利用其线程数目是2的N次幂的特性，直接通过移位运算来实现了取模操作，类似操作在`HashMap`也有体现。



## 2. `NioEventLoop`

上述`NioEventLoopGroup`在调用`newChild`时，会由`NioEventLoopGroup`创建`NioEventLoop` 对象。

`NioEventLoop` 本质上是一个线程池，内部队列维护了所有提交过来的任务，另外后台有线程定时从队列中提取任务并执行。

> NioEventLoop-->SingleThreadEventLoop-->SingleThreadEventExecutor-->AbstractScheduledEventExecutor-->AbstractEventExecutor-->AbstractExecutorService-->EventExecutor-->EventExecutorGroup

重点关注下
```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, int maxPendingTasks,
                                        RejectedExecutionHandler rejectedHandler) {
        super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = Math.max(16, maxPendingTasks);
    this.executor = ThreadExecutorMap.apply(executor, this);
    taskQueue = newTaskQueue(this.maxPendingTasks);
    rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```
`SingleThreadEventExecutor`主要作用：

1. **`executor`就是之前`NioEventLoopGroup`创建的`ThreadPerTaskExecutor`包装之后的对象**
2. **`taskQueue`队列是用来存储提交过来的任务**
3. **`rejectedExecutionHandler`是无法处理任务时的拒绝策略。**


```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                EventLoopTaskQueueFactory queueFactory) {
    super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
            rejectedExecutionHandler);
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    final SelectorTuple selectorTuple = openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}

private SelectorTuple openSelector() {
    //1.创建默认的selector对象
    final Selector unwrappedSelector;
    try {
        unwrappedSelector = provider.openSelector();
    } catch (IOException e) {
        throw new ChannelException("failed to open a new selector", e);
    }

    //DISABLE_KEY_SET_OPTIMIZATION:是否禁用优化，true:则采用原生selector运行
    if (DISABLE_KEY_SET_OPTIMIZATION) {
        return new SelectorTuple(unwrappedSelector);
    }

    //3.如果maybeSelectorImplClass不属于class或者实现类没有继承selector接口，则无法替换selectkey对象

    //生成新的selectkey集合，内部采用数组实现
    final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

    //替换原生selector的selectkey对象
    selectedKeys = selectedKeySet;
    logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
    return new SelectorTuple(unwrappedSelector,
                                new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
}
```
`openSelector`主要做了以下工作：

1. **创建默认的selector对象**
2. **判断原生的selector是否能够被替换**
3. **如果能替换则使用`SelectedSelectionKeySet`替换selector对象中的`selectedKeys`和`publicSelectedKeys`字段**

替换主要是基于性能考虑：

原生`selectedKeys`是采用的`HashSet`,极端情况下，添加操作的时间复杂度是 O(n)

`netty`则采用了数组重新生成了新的`selectedKeys`，时间复杂度始终为 O(1)



> 在看netty源码的过程中，有时候必须将相关性能优化的代码一带而过，尽管网上不少文章都对netty极致的性能优化大加赞赏，
>
> 但是从源码分析的角度来看，这不是值得关注的重点，重点在于尽快熟悉整体脉络及核心流程
>
> 相反大部分性能优化代码只会让分析者陷入困境。
>
> 就像上述的选取器`EventExecutorChooser`，不考虑性能优化及扩展，根本不需要选取器结果直接返回随机数即可，`openSelector`直接返回JDK原生对象，简单明了。
>
> 在某种程度上，如果想熟悉某个框架，选择一个初始版本应该是一个相对不错的操作。



​	


# 二. 启动

后续代码启动的逻辑，我们从下面的代码入手：
>  bootstrap.bind(30000)

最后会调用`AbstractBootStrip`的`doBind()`,`doBind`主要做了以下两件事：
1. `初始化并注册通道--initAndRegister`
2. `doBind0`

## 1. initAndRegister

### 1) 初始化通道

1.1 设置Tcp相关参数

    >  setChannelOptions(channel, newOptionsArray(), logger);

1.2 设置netty相关参数

    > setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

1.3 调用`pipeline`的`addLast`添加`ChannelInitializer`,在`ChannelInitializer`的`initChannel`中会添加`ServerBootstrapAcceptor`对象。

```java
//ServerBootstrip.cs
void init(Channel channel) {
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

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



`PendingHandlerAddedTask`继承`runnable`，本身维护了两个属性：1.`channelContext` 2.`next`

其`run`方法主要就是执行`callHandlerAdded0`，而后续又是利用`channelContext.callHandlerAdded`

最终还是调用了 `channelContext`对应的`handle.handlerAdded`

之所有不是立即执行，而是继承`runnable`,是因为触发`handlerAdd`必须等待`channel`注册成功后才能运行

### 2) 注册通道

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
    //3.如果在同一线程中，则直接执行，否则通过线程执行
    //主要是为了保证I/O事件以及用户定义的I/O事件处理逻辑在一个线程中处理
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

我们关注下执行部分，执行分为以下部分
1. 执行任务：eventLoop.execute 
2. 任务执行逻辑：register0

#### 1) `eventLoop`执行
`eventLoop.execute``继续跟进至``SingleThreadEventLoop.execute`

`loop`的执行不是简单开线程执行，而是将所有任务扔到队列中，然后利用线程从队列中取出执行

```java
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    //将任务添加进队列
    addTask(task);
    if (!inEventLoop) {
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
            }
            if (reject) {
                reject();
            }
        }
    }

    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}

private void startThread() {
    //加锁启动线程
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();
                success = true;
            } finally {
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
}
```

既然有生产任务的地方，就要有消费任务的地方，`doStartThread`就是 主要从队列中消费任务执行，主体逻辑如下：
```java
 private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            //dosomething
            this.run();
    })
 }
```
此处的`executor`实际运行时会调用前面我们初始化 生成的`ThreadPerTaskExecutor`对象，即每次执行都是new一个Thread执行
`this.run` 实现在`NioEventLoop`中

```java
 protected void run() {
        int selectCnt = 0;
       for (;;) {
            try {
                int strategy;
                try {
                    //1. 选择合适的操作策略，是直接调用selectNow,还是直接处理队列任务
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier,hasTasks());
  
                  
                selectCnt++;
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                if (ioRatio == 100) {
                    //io事件执行 比例达到100时
                    try {
                        //1.有io事件还是继续处理io
                        if (strategy > 0) {
                            processSelectedKeys();
                        }
                    } finally {
                        //2.确保会执行一次完整的非io任务(不限时)
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    //如果io执行比没有达到100，但此时有io事件，strategy=发生的事件数，由select返回
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        //确保会执行一次非Io任务(根据io占比算出一个超时时间，超过时间则终止)
                        final long ioTime = System.nanoTime() - ioStartTime;
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
                }

                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                    selectCnt = 0;
                }
            } catch (CancelledKeyException e) {
                // Harmless exception - log anyway
                if (logger.isDebugEnabled()) {
                    logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                            selector, e);
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
 }
```
重点关注下`run`方法的执行流程：

​	不管io执行占比如何，只要有io事件都会优先执行io事件。

1. 如果io事件占比达到100%,则后续执行一次完整的非io任务(无超时限制)
2. 如果io时间没有达到100%，且有IO发生，则处理完Io事件后，执行一次半完整的非IO任务(有超时限制)
3. 尽力而为

`processSelectedKeys`可以采用原生`select`或优化后的`select`执行

```
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

我们关注下`processSelectedKeysOptimized`的执行逻辑：

1. 获取selectkey
2. 根据selectkey获取channel，并处理channel
3. 判断是否需要继续轮询

继续跟踪至`processSelectedKey`：

1. 如果是OP_CONNECT事件，则调用AbstractNioChannel.unsafe.finishConnect
2. 入股是OP_WRITE，则调用`forceFlush`
3. 如果是OP_READ，则调用read

####  2) `register0`

`register0`虽然定义在`AbstractChannel`中，但是实际执行是在一个新的线程中

整体代码如下：

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

### 

## 2. 绑定

绕了一大圈，继续回到`AbstractBootstrip`中，这里好像没什么太多的流程

```java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

## 

参考资料：
[Netty源码分析(前言, 概述及目录)](https://www.cnblogs.com/xiangnan6122/p/10202191.html)
[自顶向下深入分析Netty](https://www.jianshu.com/nb/6812432)

[Netty源码分析系列之NioEventLoop的创建与启动](https://zhuanlan.zhihu.com/p/98680222)

[Netty 入门源码分析](https://www.cnblogs.com/rickiyang/p/12562408.html)

[莫那·鲁道](https://www.cnblogs.com/stateis0/category/1206890.html)