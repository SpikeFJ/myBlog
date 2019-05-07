在分析Tomcat各个组件前需要了解下Tomcat组件的生命周期、Tomcat中的几个主要组件如Server、Service、Connector、Host都是继承了Lifecycle接口，该接口主要分为init、start、stop等状态。

真正执行时的顺序则是StandardServer的init、然后是子容器(Service)的init，等所有的子容器的init方法都执行完成后才会执行StandardServer的start。

# 初始化
```
  super();

        globalNamingResources = new NamingResourcesImpl();
        globalNamingResources.setContainer(this);

        if (isUseNaming()) {
            namingContextListener = new NamingContextListener();
            addLifecycleListener(namingContextListener);
        } else {
            namingContextListener = null;
        }
```

---
# init
- 获取Catalina的ParentClassloader，也就是shareClassLoader,通过shareClassLoader加载容器里面的jar包。**在StandardContext中会调用jar检测**
- 调用所有service的init方法
---

# start
```
 fireLifecycleEvent(CONFIGURE_START_EVENT, null);
setState(LifecycleState.STARTING);

globalNamingResources.start();

// Start our defined Services
synchronized (servicesLock) {
    for (int i = 0; i < services.length; i++) {
        services[i].start();
    }
}
```        
---


# await
- port=-2
    - 直接return(为Embeded服务)
- port=-1
    - awaitThread=当前线程，每隔1s检测stopAwait变量
- port = 8005
    - 新建ServerSocket，监听8005端口，阻塞在accept方法上
    - accept，超时时间10s
    - 默认shutdown命令=“SHUTDOWN”，但为了防止DoS，如果shutdown命令>1024,截取1024个字符,做匹配。
***

# stop
- 调用StandardService的stop
- stopAwait，关闭监听socket
***

# TODO
* JMX
* PropertyChangeSupport
* init中ExtensionValidator.addSystemResource是什么？

