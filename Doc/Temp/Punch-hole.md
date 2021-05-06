# 一. 预备知识：

* 内网访问外网时，路由器/防火墙会将内网机器(**源IP端口**)的ip和端口改写，包装成自己的IP+随机端口

* 路由器/防火墙在包装Ip和端口时，不管 **目的IP端口** 是什么，只要来源ip端口一致则包装成成相同的IP端口

举例：

> Host1通过本地端口12345访问 XXXXXX:4567,如果经过路由器改写后的源IP端口是：1.1.1.1:5678

> 那么，Host1通过本地端口12345访问 YYYYYY:6789,出来的源IP端口也会是：1.1.1.1:5678

# 二. 流程分析

punch hole 程序分为两部分：服务端，客户端

下面是网络示意图：

![image-20210506105414108](https://i.loli.net/2021/05/06/TR3wa9PGcEJnUZH.png)

>Router是路由器
>
>Server是挖洞服务端程序部署机器
>
>Client是挖洞客户端部署机器



步骤梳理：

1. 客户端程序启动时，和服务端建立通讯连接（内网可以自由访问外网）；服务端将当前已经连接的客户端列表发送给初次上线的`client`，这样，服务端和客户端都维护了当前在线的`client`列表（包含用户名，ip端口等信息，ip端口则是被路由器改写过的）

2. 假设`clientA`想和`clientB`建立连接，此时`clientA`尽管知道`clientB`的外网信息（步骤1已经在客户端维护了当前所有在线列表），但直接连接`clientB`，会被`RouterB`丢弃。因为这属于不请自来的请求，基于安全考虑，大都数`Router`都会执行丢弃。

   所以我们需要由`ClientB`发起请求，当然该请求会被RouterA丢弃，**但是该请求的副作用就是RouterA发送给ClientB的后续请求就不会被`RouterB`丢弃了。**

3. 此时`ClientA`再次向ClientB发送信息，则不会被`RouterB`丢弃



流程图如下：

![image-20210506111728648](https://i.loli.net/2021/05/06/5aSYPJ8gjxVkXMt.png)





参考资料

- [ P2P NAT穿越](https://blog.csdn.net/daitu3201/article/details/80407323?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-18.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-18.control)
- -[P2P技术详解](http://www.52im.net/thread-50-1-1.html)

