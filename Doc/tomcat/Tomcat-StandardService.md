一个Server中包含多个Service，Service中主要有以下重要属性：
* MapperListener
* Mapper
* Engine
* Connector

一个Service由多个Connector和一个Engine组成
# init
```
  protected void initInternal() throws LifecycleException {

        super.initInternal();

        //引擎初始化
        if (engine != null) {
            engine.init();
        }

        //executor初始化
        for (Executor executor : findExecutors()) {
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled) executor).setDomain(getDomain());
            }
            executor.init();
        }

        // 映射监听器初始化
        mapperListener.init();

        //连接器初始化
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                connector.init();
            }
        }
    }
```