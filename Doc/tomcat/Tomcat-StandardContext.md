复习下之前的容器启动步骤
* StandardEngie调用基类的start方法，开线程启动子容器
* StandardHost启动成功，会触发HostConfig的监听事件
* HostConfig监听处理会加载、实例化StandardContext对象，并执行init、start方法

今天分析下StandardContext对象
# 初始化
```java
public StandardContext() {

    super();
    pipeline.setBasic(new StandardContextValve());
    broadcaster = new NotificationBroadcasterSupport();
    // Set defaults
    if (!Globals.STRICT_SERVLET_COMPLIANCE) {
        // Strict servlet compliance requires all extension mapped servlets
        // to be checked against welcome files
        resourceOnlyServlets.add("jsp");
    }
}
```

# init
主要是一些资源的初始化，先略过
```java
protected void initInternal() throws LifecycleException {
    super.initInternal();

    // Register the naming resources
    if (namingResources != null) {
        namingResources.init();
    }

    // Send j2ee.object.created notification
    if (this.getObjectName() != null) {
        Notification notification = new Notification("j2ee.object.created",
                this.getObjectName(), sequenceNumber.getAndIncrement());
        broadcaster.sendNotification(notification);
    }
}
```

# start
* 资源的初始化及启动
* WebappLoader的初始化
* Rfc6265CookieProcessor初始化
* 初始化字符集映射
* 
