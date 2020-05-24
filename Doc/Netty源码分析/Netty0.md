`netty` 目前应该是 java 世界最主流的通讯连接框架。

 `Dubbo`，`RocketMq`等一系列知名框架也都采用 `netty`作为其底层通讯基础。

所以还是有必要了解下`netty`的内部实现。

***

在探究`netty`内部实现前，需要有一些前置条件

* 熟悉/了解 `TCP/IP`基础。
* 熟悉 java Nio 基础。

本系列都采用 `netty 4.1`作为分析对象。


1. ```
   NioEventLoopGroup 继承 MultithreadEventLoopGroup
   
   MultithreadEventLoopGroup 初始化完成 默认线程数目的设置：取io.netty.eventLoopThreads，没有则用当前2倍的处理器数目
   
   ```





