最近发现了一款比较好的内网映射工具-[`lanproxy`](https://github.com/ffay/lanproxy),

采用 java 开发，源码质量很高，而且作者源码开放后的商业化操作也值得借鉴。

至于内网映射是什么，有什么应用场景，这里就不介绍了。`lanproxy`官网都有介绍

# 一. 结构图

![image-20200810123158046](https://i.loli.net/2020/08/10/DCaVH893ulAOqTX.png)连接流程图：

![image-20200810150605359](https://i.loli.net/2020/08/10/SZyABEHaTrnObhG.png)

传输流程图略：

大体传输路径就是 30002转发程序，找到步骤6中的连接，最后转发给最终的实际服务。

# 二. proxy-server

## 1. config-`properties`

![image-20200810130549617](https://i.loli.net/2020/08/10/EL1TVwDXiveBCYu.png)

## 2. 源码分析

proxy-`server`分为两个监听模块：`ProxyServerContainer`和`WebConfigContainer`

1. `ProxyServerContainer`负责和`proxy-client`通讯
2. `WebConfigContainer`负责对外提供`web`配置页面



### 2.1. 启动`ProxyServerContainer`

`ProxyServerContainer`负责两部分连接：

1. `proxy-client`通讯连接。默认监听端口4900，对应的处理类是：`ServerChannelHandler`
2. 转发端口连接（如用户通过web配置了公网端口30002，则需要开启30002端口监听请求），对应代码中`startUserPort`，对应的处理类是：`UserChannelHandler`

由于随着web配置的公网端口的改变，`ProxyServerContainer`需要能够实时的启动相关端口的监听，

所以这里采用了观察者模式：`ProxyServerContainer`既实现了`Container`，也实现了`ConfigChangedListener`，当配置参数变更时，能够做出实时响应。

![image-20200810131135454](https://i.loli.net/2020/08/10/MgA6he9xXEOStf7.png)

### 2.2. 启动``WebConfigContainer``

![image-20200810132726108](https://i.loli.net/2020/08/10/TarE6hcFb4msLAY.png)

`HttpRequestHandler`作为`web`所有的请求入口，在其`channelRead0`中对请求进行静态、动态路由转发

具体的动态路由转发，由`RouteConfig`配置

，我们主要关注下面的路由处理`/config/update`

![image-20200810133109840](https://i.loli.net/2020/08/10/OKCrqGnHvyXtuQB.png)

当用户配置公网端口转发时，会交由该路由处理程序，注意此处的`ProxyConfig.getInstance().update(config);`,会通知`ProxyServerContainer`及时开启相关监听端口

### 2.3. `proxy-client`

`proxy-client`启动后连接`proxy-server`4900端口，

连接成功后发送`clientkey`,即客户端密钥给-`proxyserver`验证其身份，

由于`proxy-client`添加了`IdleCheckHandler`，会和`proxy-server`一直保持心跳

![image-20200810134553877](https://i.loli.net/2020/08/10/8cCguOZLQB1Y5af.png)

下面是服务端接受到认证信息的处理流程,参见`ServerChannelHandler`：

![image-20200810135457241](https://i.loli.net/2020/08/10/pwaOdiJcF173MEb.png)

### 2.4. 访问转发端口`UserChannelHandler`

首先关注下`channelActive`方法，对应着上述流程图中步骤3：

![image-20200810151626722](https://i.loli.net/2020/08/10/Z5EtTPezyN1cUwg.png)



后续数据传输时，关注下`channelRead0`:

![image-20200810151838113](https://i.loli.net/2020/08/10/SzaFmZKEAQ8UiGf.png)

现在可以实际测试下转发效果了，由于我们在之前配置了公网30002作为转发端口，转发至127.0.0.1:8080

可以在proxy-client所在机器上开一个TCP监听8080

![image-20200810140439685](https://i.loli.net/2020/08/10/jKU5vHTLQugwdt9.png)