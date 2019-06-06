# 容器

在分析StandardEngine前先了解下Tomcat的容器的概念。

1. Server是Service的容器
2. Service是Connector和Container的容器
3. Container自包含:Engine-->Host-->Context-->Wrapper
4. Container继承LifeCycle接口，LifeCycle
5. 每个容器都含有PipeLine对象

## LifeCycle
lifeCycle是所有带有生命周期的容器类组件的公共接口，并包含lifeListener接口的集合，典型的观察者模式。整体结构如下：
> init-->start-->stop-->destroy

在整体流程确定的基础上预留了供观察者监听的事件类型：
> before_init、after_init

> before_start、start、after_start

>before_stop、stop、after_stop

>before_destroy、after_destroy

>periodic、configure_start

* LifecycleBase继承lifeCycle，实现了观察者模式的基础框架。我们以init为例，start、stop、destroy类似，主要方法如下：
``` java
   private final List<LifecycleListener> lifecycleListeners = new CopyOnWriteArrayList<>();

   //通知观察者
   protected void fireLifecycleEvent(String type, Object data) {
        LifecycleEvent event = new LifecycleEvent(this, type, data);
        for (LifecycleListener listener : lifecycleListeners) {
            listener.lifecycleEvent(event);
        }
   }
  
   //以init方法为例
   public final synchronized void init() throws LifecycleException {
    //检查当前容器状态，不满足则抛出异常
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }

    try {
        //切换状态为初始化中、并调用fireLifecycleEvent通知相关监听器(观察者)
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        //真正的init逻辑交由子类实现
        initInternal();
        //切换状态为初始化完成、并调用fireLifecycleEvent通知相关监听器(观察者)
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {
        handleSubClassException(t, "lifecycleBase.initFail", toString());
    }
}

```
---
StardardEngine是容器启动的入口，不过没有重写initInternal和startInternal方法。
所以调用的Containerbase的init和start
```java
Container children[] = findChildren();
    List<Future<Void>> results = new ArrayList<>();
    for (int i = 0; i < children.length; i++) {
        results.add(startStopExecutor.submit(new StartChild(children[i])));
    }

    boolean fail = false;
    for (Future<Void> result : results) {
        try {
            result.get();
        } catch (Exception e) {
            log.error(sm.getString("containerBase.threadedStartFailed"), e);
            fail = true;
        }

    }
```
可以看到开线程遍历所有的子容器的start方法，在调用子容器的start之前会判断状态调用init方法


```java
 protected synchronized void startInternal() throws LifecycleException {

        
        logger = null;
        getLogger();
        //集群启动
        Cluster cluster = getClusterInternal();
        if (cluster instanceof Lifecycle) {
            ((Lifecycle) cluster).start();
        }
        //角色启动
        Realm realm = getRealmInternal();
        if (realm instanceof Lifecycle) {
            ((Lifecycle) realm).start();
        }

        //启动子容器，
        //此处会触发hostconfig对象的start方法
        Container children[] = findChildren();
        List<Future<Void>> results = new ArrayList<>();
        for (int i = 0; i < children.length; i++) {
            results.add(startStopExecutor.submit(new StartChild(children[i])));
        }

        boolean fail = false;
        for (Future<Void> result : results) {
            try {
                result.get();
            } catch (Exception e) {
                log.error(sm.getString("containerBase.threadedStartFailed"), e);
                fail = true;
            }

        }
        if (fail) {
            throw new LifecycleException(
                    sm.getString("containerBase.threadedStartFailed"));
        }

        // 启动pipeline
        if (pipeline instanceof Lifecycle)
            ((Lifecycle) pipeline).start();

        setState(LifecycleState.STARTING);

        //启动后台线程
        threadStart();

    }
    
```

threadStart启动的后台进程如下:
```java
protected class ContainerBackgroundProcessor implements Runnable {

    @Override
    public void run() {
        Throwable t = null;
        String unexpectedDeathMessage = sm.getString(
                "containerBase.backgroundProcess.unexpectedThreadDeath",
                Thread.currentThread().getName());
        try {
            while (!threadDone) {
                try {
                    Thread.sleep(backgroundProcessorDelay * 1000L);
                } catch (InterruptedException e) {
                    // Ignore
                }
                if (!threadDone) {
                    processChildren(ContainerBase.this);
                }
            }
        } catch (RuntimeException|Error e) {
            t = e;
            throw e;
        } finally {
            if (!threadDone) {
                log.error(unexpectedDeathMessage, t);
            }
        }
    }

    protected void processChildren(Container container) {
        ClassLoader originalClassLoader = null;

        try {
            if (container instanceof Context) {
                Loader loader = ((Context) container).getLoader();
                // Loader will be null for FailedContext instances
                if (loader == null) {
                    return;
                }

                // Ensure background processing for Contexts and Wrappers
                // is performed under the web app's class loader
                originalClassLoader = ((Context) container).bind(false, null);
            }
            container.backgroundProcess();
            Container[] children = container.findChildren();
            for (int i = 0; i < children.length; i++) {
                if (children[i].getBackgroundProcessorDelay() <= 0) {
                    processChildren(children[i]);
                }
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            log.error("Exception invoking periodic operation: ", t);
        } finally {
            if (container instanceof Context) {
                ((Context) container).unbind(false, originalClassLoader);
            }
        }
    }
}
```
后续参见[tomcat](Tomcat-HostConfig)
