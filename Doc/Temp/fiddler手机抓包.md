参考:

[Fiddler使用总结一（使用Fiddler捕获手机所有http/https通信）](https://www.cnblogs.com/iovec/p/fiddler_01.html)

总结下：

1. PC端链接Wifi1，打开fiddler，设置监听端口，允许接入，https等配置
2. 手机端接入相同的Wifi1，并且手动设置代理，输入步骤1中PC端的ip和监听端口
3. 注意1：PC端防火墙需要关闭，或者设置相应的规则放行
4. 注意2：部分路由器或路由器系统有【用户隔离】选项，需要关闭。

> 可以参考下面的参考资料，
发现的源头在于发现局域网之间无法ping通，解决：在路由器管理设置里，找到安全设置》用户隔离，状态改为关闭。本人用的openwrt，默认是开启的，所以在这一步花费了不少时间。

参考[Fiddler抓包手机无法访问](https://testerhome.com/topics/2651
)
