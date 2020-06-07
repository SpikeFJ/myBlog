# 一. `NioEventLoopGroup`

首先简单看下`NioEventLoopGroup`的类继承结构

​	![](D:\gitResource\myBlog\Resource\netty\netty0_2020-06-06_15-06-50.png)

## 1. `NioEventLoopGroup`

`NioEventLoopGroup`的相关构造函数如下：

```java
//默认线程数0
public NioEventLoopGroup() {
    this(0);
}
//默认线程执行器null
public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
}
//默认selector提供器：SelectorProvider.provider()
public NioEventLoopGroup(ThreadFactory threadFactory) {
    this(0, threadFactory, SelectorProvider.provider());
}
//默认选择策略工厂类：DefaultSelectStrategyFactory.INSTANCE
public NioEventLoopGroup(
    int nThreads, ThreadFactory threadFactory, final SelectorProvider selectorProvider) {
    this(nThreads, threadFactory, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}
//默认拒绝策略工厂类：RejectedExecutionHandlers.reject()
public NioEventLoopGroup(int nThreads, ThreadFactory threadFactory,
                         final SelectorProvider selectorProvider, final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, threadFactory, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}


```

可以看到`NioEventLoopGroup`主要通过构造函数重载提供了各种默认机制。

## `2.MultithreadEventLoopGroup`

1. 静态初始化块：

```java
//获取
static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
        "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}

```

这里主要是获取根据`io.netty.eventLoopThreads`配置获取默认的线程数，

如果没有设置该参数，则默认是cpu处理器*2



2. 构造函数

   构造函数主要判断如果线程数为0，则采用`DEFAULT_EVENT_LOOP_THREADS`来作为线程数。

   所有`NioEventLoopGroup`中如果设置了线程数，则采用设置的值；否则则采用`DEFAULT_EVENT_LOOP_THREADS`作为线程数。

## 3. `MultithreadEventExecutorGroup`

经过前面层层调用，终于看到了曙光：

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

    //创建一个线程选取器
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

总结下这里构造函数主要做了以下工作：

1. 创建一个线程执行器`ThreadPerTaskExecutor`，该执行器的作用在于：

   后续启动`NioEventLoopGroup`组中的Loop对象时，不能串行执行，所以会利用该执行器开多个线程同时启动

2. **通过`newChild`方法初始化`loop`对象，该方法由`NioEventLoopGroup`对象实现**

3. 创建一个选取器，该选取器的作用在一个`NioEventLoopGroup`维护着多个`NioEventLoop`对象，当连接到达时，应该使用哪一个`NioEventLoop`对象来处理？这就是选取器：`EventExecutorChooser`要负责的。

4. 给每一个`loop`对象添加终止事件

5. 将所有`loop`对象添加到一个只读数组中



# 二.`NioEventLoopGroup`

下面我们看下`NioEventLoop`是如何生成的，创建入口在`NioEventLoopGroup`：

```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}
```

`NioEventLoop`的类层级结构如下：

![](D:\gitResource\myBlog\Resource\netty\netty0_2020-06-06_15-38-07.png)

因为此处构造函数调用都是先通过`super`调用基类，再子类。

所以和分析`NioEventLoopGroup`分析顺序略有不同，我们先分析基类（没有实质逻辑的基类暂不分析），再子类。

## `1.AbstractEventExecutor`

因为`NioEventLoop`本质上还是一个执行器，所以它继承了`AbstractEventExecutor`

而构造函数中也很简单，将`group`对象做为`parent`保存，

```java
protected AbstractEventExecutor(EventExecutorGroup parent) {
    this.parent = parent;
}
```

## 2. `SingleThreadEventExecutor`

从命名可以看出来，该类是在一个线程中执行所有提交过来的任务，

既然是单线程执行，那必须具备下列元素：

1.一个执行线程 

2.一个任务队列

```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                    RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
    this.executor = ThreadExecutorMap.apply(executor, this);
    this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
    this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```

`SingleThreadEventExecutor`主要作用：

1. **`executor`就是之前`NioEventLoopGroup`创建的`ThreadPerTaskExecutor`包装之后的对象**
2. **`taskQueue`队列是用来存储提交过来的任务**
3. **`rejectedExecutionHandler`是无法处理任务时的拒绝策略。**

## 3. `NioEventLoop`

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
```

我们重点关注下`openSelector()`方法

```java
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

原生`selectedKeys`采用的`HashSet`,极端情况下，添加操作的时间复杂度是 O(n)

`netty`则采用了数组重新生成了新的`selectedKeys`，时间复杂度始终为 O(1)



> 在看`netty`源码的过程中，有时候必须将相关性能优化的代码一带而过，
>
> 尽管网上不少文章都对netty极致的性能优化大加赞赏，
>
> 但是从源码分析的角度来看，这不是值得关注的重点，重点在于尽快熟悉整体脉络及核心流程
>
> 相反大部分性能优化代码只会让分析者陷入困境。
>
> 就像上述的选取器`EventExecutorChooser`，不考虑性能优化及扩展，根本不需要选取器结果直接返回随机数即可，`openSelector`直接返回JDK原生对象，简单明了。
>
> 在某种程度上，如果想熟悉某个框架，选择一个初始版本应该是一个相对不错的操作。



# 三. 总结

1. `NioEventLoopGroup`内部维护了多个`NioEventLoop`，如何选择相应的`NioEventLoop`,则是通过内部的线程选择器来决定的

2. `NioEventLoop`本质上是一个线程执行器

3. `NioEventLoop`内部维护了一个`selector`多路复用器


***

# 四. `NioEventLoop`执行

上面我们已经讨论了作为一个执行器，`loop`必须有队列，执行线程，

另外还需要一个字段区分当前执行器的状态，

```java
//执行器状态，默认未开始 
private volatile int state = ST_NOT_STARTED;
//1.未开始
private static final int ST_NOT_STARTED = 1;
//2.已开始
private static final int ST_STARTED = 2;
//3.正在关闭
private static final int ST_SHUTTING_DOWN = 3;
//4.已关闭
private static final int ST_SHUTDOWN = 4;
//5.已停止
private static final int ST_TERMINATED = 5;
```



因为后续很多任务都是通过`nioEventLoop`执行任务的，所以我们关注下其`execute`方法

```java
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    //将任务添加进队列taskQueue
    addTask(task);
    if (!inEventLoop) {
        //开启线程
        startThread();
        //如果已关闭
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
	//是否要唤醒执行器，立即执行任务
    //addTaskWakesUp默认为false  immediate为true
    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}

private void startThread() {
    if (state == ST_NOT_STARTED) {
        //采用cas修改state状态为启动
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
                 //1. 默认select操作策略：如果有任务则返回调用selectNow的返回值，否则返回SelectStrategy.SELECT
                 strategy=selectStrategy.calculateStrategy(selectNowSupplier,hasTasks());
  switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
                    default:
                    }
                  
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
                    
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) {
                    selectCnt = 0;
                }
            } catch (CancelledKeyException e) {

            } catch (Throwable t) {
                handleLoopException(t);
            }
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



可以看到`run`方法主要做了两件事：1. 处理IO(`processSelectedKeys`) 2.处理普通任务(`runAllTasks`)



## 1. 处理IO事件

`processSelectedKeys`可以采用原生`select`或优化后的`select`执行

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

我们关注下`processSelectedKeysOptimized`的执行逻辑：

1. 获取`selectkey`
2. 根据`selectkey`获取`channel`，并处理`channel`
3. 判断是否需要继续轮询

继续跟踪至`processSelectedKey`：

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            return;
        }
        if (eventLoop == this) {
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        //连接接入事件
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }
        //写事件
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            ch.unsafe().forceFlush();
        }
        //如果是bossGroup:Accept事件
        //如果是workGroup:read事件
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

1. 如果是`OP_CONNECT`事件，则调用`AbstractNioChannel.unsafe.finishConnect`
2. 入股是`OP_WRITE`，则调用`forceFlush`
3. 如果是`OP_READ`，则调用`read`

## 2. 处理普通提交任务

### 2.1 带超时限制的任务

首先我们关注下下面的语句：

```java
 runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
```

`ioTime`：表示执行一次`select`所耗费的时间

`(100 - ioRatio) / ioRatio`:表示非`IO`任务应占时间比例

所以`ioTime * (100 - ioRatio) / ioRatio`就是非IO任务执行允许的最长执行时间，也就是超时时间。

下面看下`runAllTasks`的具体实现

```java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }

    final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);

        runTasks ++;

        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

待续......



