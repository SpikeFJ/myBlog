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
* 资源StandardRoot(含class文件、jar包及其他资源文件)的初始化及启动
* WebappLoader的初始化，启动时会创建WebappClassLoader
* Cookie处理器Rfc6265CookieProcessor初始化
* 初始化字符集映射
* 初始化工作目录
* ExtensionValidator.validateApplication
* bindThread
* loader.start()
* Realm.start()
* 通知ContextConfig处理：fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);
> 解析Web.xml，请求映射、filter设置等操作是由ContextConfig执行的
* child.start()
* pipeline.start()
* 生成Session管理器，区分是否集群：new StandardManager/getCluster().createManager
* 将WebResourceRoot添加到ServletContext属性中
* 创建DefaultInstanceManager，用于创建Servlet及Filter
* 将StandardJarScanner添加到ServletContext属性中
* mergeParameters
* ServletContainerInitializer.onStartup()
* 启动监听器
* 检查未覆盖的http方法安全
* 启动StandardManager
* 初始化filterConfig(filterStart)
* 对于loadOnStartup>0的wrapper，调用Wrapper.load()(loadOnStartup)
* 启动ContainerBase的后台线程
* WebResourceRoot.gc()释放资源

StandardContext做的工作比较多，毕竟这部分才是用户直接打交道的。。。。<br>
好吧，一个个来
