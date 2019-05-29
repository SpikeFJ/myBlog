* 在生命周期管理中，LifecycleBase采用了**CopyOnWriteArrayList**作为监听器的管理容器
* 大部分类中都采用**StringManager**作为资源管理
* Connect-Adapter-EnumSet表示SSL_ONLY，参考：Tomcat SSL配置
* NioEndpoint-->Accpter-TaskQueue-->**LinkedBlockingQueue**
* 所有容器的都继承ContainerBase，ContainerBase调用initInternal方法时**ThreadPoolExecutor**
* CompleteableFuture