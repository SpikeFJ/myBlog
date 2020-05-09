xxl-job提供了下面几种执行器部署方式
> 1. xxl-job-executor-sample-frameless
> 2. xxl-job-executor-sample-jboot
> 3. xxl-job-executor-sample-jfinal
> 4. xxl-job-executor-sample-nutz
> 5. xxl-job-executor-sample-spring
> 6. xxl-job-executor-sample-springboot

因为主要是关注原理性的东西，所以排除掉第三方框架的影响，选择`xxl-job-executor-sample-frameless`项目作为分析的重点。


首先注册已有的`jobHandler`
```java
XxlJobExecutor.registJobHandler("demoJobHandler", new DemoJobHandler());
XxlJobExecutor.registJobHandler("shardingJobHandler", new ShardingJobHandler());
XxlJobExecutor.registJobHandler("httpJobHandler", new HttpJobHandler());
XxlJobExecutor.registJobHandler("commandJobHandler", new CommandJobHandler());
```

然后执行`  xxlJobExecutor.start();`,
```java
public void start() throws Exception {
    //1.初始化执行器日志路径
    XxlJobFileAppender.initLogPath(logPath);

    //2.保存配置文件的调度中心地址和访问token，多个调度中心地址以，分隔
    initAdminBizList(adminAddresses, accessToken);

    //3.启动日志定时清理程序
    JobLogFileCleanThread.getInstance().start(logRetentionDays);

    //4.启动回调函数执行线程
    TriggerCallbackThread.getInstance().start();

    //5.通知调度中心
    port = port>0?port: NetUtil.findAvailablePort(9999);
    ip = (ip!=null&&ip.trim().length()>0)?ip: IpUtil.getIp();
    initRpcProvider(ip, port, appName, accessToken);
}
```
`xxl-job-executor.properties`配置文件如下：
```properties
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin

### xxl-job executor address
xxl.job.executor.appname=xxl-job-executor-sample
xxl.job.executor.ip=
xxl.job.executor.port=9994

### xxl-job, access token
xxl.job.accessToken=

### xxl-job log path
xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler
### xxl-job log retention days
xxl.job.executor.logretentiondays=30
```


# 一.`initRpcProvider`
```java
private void initRpcProvider(String ip, int port, String appName, String accessToken) throws Exception {
    // init, provider factory
    String address = IpUtil.getIpPort(ip, port);
    Map<String, String> serviceRegistryParam = new HashMap<String, String>();
    //appName是每个执行器的唯一标识
    serviceRegistryParam.put("appName", appName);
    //address如果配置文件有配置，否则采用本地ip+默认端口9999
    serviceRegistryParam.put("address", address);

    xxlRpcProviderFactory = new XxlRpcProviderFactory();

    xxlRpcProviderFactory.setServer(NettyHttpServer.class);
    xxlRpcProviderFactory.setSerializer(HessianSerializer.class);
    xxlRpcProviderFactory.setCorePoolSize(20);
    xxlRpcProviderFactory.setMaxPoolSize(200);
    xxlRpcProviderFactory.setIp(ip);
    xxlRpcProviderFactory.setPort(port);
    xxlRpcProviderFactory.setAccessToken(accessToken);
    //配置注册处理对象
    xxlRpcProviderFactory.setServiceRegistry(ExecutorServiceRegistry.class);
    //配置注册参数：appName和address
    xxlRpcProviderFactory.setServiceRegistryParam(serviceRegistryParam);

    // add services
    xxlRpcProviderFactory.addService(ExecutorBiz.class.getName(), null, new ExecutorBizImpl());

    // start
    xxlRpcProviderFactory.start();
}
```

1. `XxlRpcProviderFactory`的`start`主要代码如下
```java
//生成序列化对象
this.serializerInstance = (Serializer)this.serializer.newInstance();
this.serviceAddress = IpUtil.getIpPort(this.ip, this.port);
//生成NettyServer对象，开启监听
this.serverInstance = (Server)this.server.newInstance();
//设置启动后的回调函数
this.serverInstance.setStartedCallback(new BaseCallback() {
    public void run() throws Exception {
        if (XxlRpcProviderFactory.this.serviceRegistry != null) {
            //生成 注册工厂对象-->ExecutorServiceRegistry
            XxlRpcProviderFactory.this.serviceRegistryInstance = (ServiceRegistry)XxlRpcProviderFactory.this.serviceRegistry.newInstance();
            //根据传递参数{appName,address}启动 注册工厂对象
            XxlRpcProviderFactory.this.serviceRegistryInstance.start(XxlRpcProviderFactory.this.serviceRegistryParam);

            //如果工厂包含服务对象，则进行服务注册，该步骤
            if (XxlRpcProviderFactory.this.serviceData.size() > 0) {
                XxlRpcProviderFactory.this.serviceRegistryInstance.registry(XxlRpcProviderFactory.this.serviceData.keySet(), XxlRpcProviderFactory.this.serviceAddress);
            }
        }

    }
});
this.serverInstance.setStopedCallback(new BaseCallback() {
    public void run() {
        if (XxlRpcProviderFactory.this.serviceRegistryInstance != null) {
            if (XxlRpcProviderFactory.this.serviceData.size() > 0) {
                XxlRpcProviderFactory.this.serviceRegistryInstance.remove(XxlRpcProviderFactory.this.serviceData.keySet(), XxlRpcProviderFactory.this.serviceAddress);
            }

            XxlRpcProviderFactory.this.serviceRegistryInstance.stop();
            XxlRpcProviderFactory.this.serviceRegistryInstance = null;
        }

    }
});
this.serverInstance.start(this);

。。。。
 public static class ExecutorServiceRegistry extends ServiceRegistry {
        @Override
        public void start(Map<String, String> param) {
            // start registry
            ExecutorRegistryThread.getInstance().start(param.get("appName"), param.get("address"));
        }
 }
```

2. `ExecutorRegistryThread`分析

    `ExecutorRegistryThread`启动一个后台轮询线程，该线程逻辑如下

    1. 遍历所有的调度中心，针对每个调度中心执行注册
    ```java
    RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appName, address);
    for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
            ReturnT<String> registryResult = adminBiz.registry(registryParam);
    }
    ```
    2.注册代码参考`AdminBizClient`：
    ```java
    public ReturnT<String> registry(RegistryParam registryParam) {
        return XxlJobRemotingUtil.postBody(addressUrl + "api/registry", accessToken, registryParam, 3);
    }
    ```
    3.【`xxl-job-admin`】模块的`JobApiController`中
    ```java
    @RequestMapping("/registry")
     public ReturnT<String> registry(HttpServletRequest request, @RequestBody(required = false) String data) {
        // 1.校验accessToken
        validAccessToken(request);

        //2. 解析EXECUTOR，appName和address
        RegistryParam registryParam = (RegistryParam) parseParam(data, RegistryParam.class);

        //3. 调用adminBiz注册
        return adminBiz.registry(registryParam);
    }
    ```
    4.【`xxl-job-admin`】模块的`AdminBizImpl`
    ```java
    public ReturnT<String> registry(RegistryParam registryParam) {

        // 1.检验参数
        if (!StringUtils.hasText(registryParam.getRegistryGroup())
                || !StringUtils.hasText(registryParam.getRegistryKey())
                || !StringUtils.hasText(registryParam.getRegistryValue())) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "Illegal Argument.");
        }
        //2.根据RegistryParam更新xxl_job_registry对应记录的更新时间
        int ret = xxlJobRegistryDao.registryUpdate(registryParam.getRegistryGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue(), new Date());
        if (ret < 1) {
            // 3.若没有该记录，则插入记录
            xxlJobRegistryDao.registrySave(registryParam.getRegistryGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue(), new Date());

            // fresh
            freshGroupRegistryInfo(registryParam);
        }
        return ReturnT.SUCCESS;
    }
    ```
**总结：**

   至此上篇留下的第一个疑问解决：执行器在启动时，主动通知调度中心已上线，调度中心更新`xxl_job_registry`,

   如果还有印象的话，调度中心`JobRegistryMonitorHelper`作为一个后台线程，会一直查询`xxl_job_registry`表，并将表中的机器信息更新到执行器`xxl_job_group`的`address_list`地段上

   所以这里就解释了为什么调度中心调度时可以知道一共有哪些机器支持被调度。

# 二. 处理调度中心传递的信息

首先从`NettyHttpServer`着手，看到如下代码：
```java
bootstrap.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)).childHandler(new ChannelInitializer<SocketChannel>() {
    public void initChannel(SocketChannel channel) throws Exception {
        channel.pipeline()
        .addLast(new ChannelHandler[]{new IdleStateHandler(0L, 0L, 90L, TimeUnit.SECONDS)})
        .addLast(new ChannelHandler[]{new HttpServerCodec()})
        .addLast(new ChannelHandler[]{new HttpObjectAggregator(5242880)})
        .addLast(new ChannelHandler[]{new NettyHttpServerHandler(xxlRpcProviderFactory, serverHandlerPool)});
    }
}).childOption(ChannelOption.SO_KEEPALIVE, true);
```

继续跟踪`NettyHttpServerHandler`,在`process`中可以看到是交由`xxlRpcProviderFactory.invokeService`处理传递过来的`XxlRpcRequest`：
```java
public XxlRpcResponse invokeService(XxlRpcRequest xxlRpcRequest) {
    XxlRpcResponse xxlRpcResponse = new XxlRpcResponse();
    //1.请求id需要回写
    xxlRpcResponse.setRequestId(xxlRpcRequest.getRequestId());
    //2.生成服务key
    String serviceKey = makeServiceKey(xxlRpcRequest.getClassName(), xxlRpcRequest.getVersion());
    Object serviceBean = this.serviceData.get(serviceKey);
    if (serviceBean == null) {
        xxlRpcResponse.setErrorMsg("The serviceKey[" + serviceKey + "] not found.");
        return xxlRpcResponse;
    } else if (System.currentTimeMillis() - xxlRpcRequest.getCreateMillisTime() > 180000L) {
        xxlRpcResponse.setErrorMsg("The timestamp difference between admin and executor exceeds the limit.");
        return xxlRpcResponse;
    } else if (this.accessToken != null && this.accessToken.trim().length() > 0 && !this.accessToken.trim().equals(xxlRpcRequest.getAccessToken())) {
        xxlRpcResponse.setErrorMsg("The access token[" + xxlRpcRequest.getAccessToken() + "] is wrong.");
        return xxlRpcResponse;
    } else {
        try {
            //serviceClass---->ExecutorBizImpl
            Class<?> serviceClass = serviceBean.getClass();
            String methodName = xxlRpcRequest.getMethodName();
            Class<?>[] parameterTypes = xxlRpcRequest.getParameterTypes();
            Object[] parameters = xxlRpcRequest.getParameters();
            Method method = serviceClass.getMethod(methodName, parameterTypes);
            method.setAccessible(true);
            Object result = method.invoke(serviceBean, parameters);
            xxlRpcResponse.setResult(result);
        } catch (Throwable var11) {
            logger.error("xxl-rpc provider invokeService error.", var11);
            xxlRpcResponse.setErrorMsg(ThrowableUtil.toString(var11));
        }

        return xxlRpcResponse;
    }
}
```
# 三. `ExecutorBizImpl`,
最后反射生成的对象是`ExecutorBizImpl`，这里重点关注下`run`方法,其他如`idleBeat`,`kill`则略过:

1. 根据`jobid`获取对应的处理线程，如果处理线程不存在，则通过`registJobThread`生成`JobThread`，该对象是和`job`，`handler`绑定的。

2. `JobThread`的作用是首次生成时初始化`handler`对象，并且循环读取内部的待执行队列元素，通过判断是否有执行超时配置 来决定是采用`FutureTask`来执行具体的`handler`还是直接执行。

3. 上述元素执行完成后，将执行对象和结果封装到`HandleCallbackParam`,传递到`callBackQueue`队列中待处理。
