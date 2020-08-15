今天有点无聊，准备找点面试题看下，看到有一道交替打印ABC多次的并发题目，练练手

准备以下几种方式

# 一. `Lock`锁+`Condition`

![image-20200815133729390](https://i.loli.net/2020/08/15/vm7WyVZYMcGPgSe.png)

注意启动时候必须按一下顺序：

```java
//首先启动C、B,防止A线程调用唤起时，还没有线程在等待信号
new C().start();
new B().start();
new A().start();
```

# 二. `synchronized `

![image-20200815180230446](https://i.loli.net/2020/08/15/NFMzET1oiDbsOpV.png)



`synchronized`可以依靠`object`对象的`wait`和`notify`配合实现，

思路是每个线程同时持有一个本身的资源对象和下一个线程需要的资源对象，

线程A执行：1.打印 2.通知下一个线程 3. 阻塞

线程A执行：1.阻塞 2.打印 3.通知下一个线程  

线程A执行：1.阻塞 2.打印 3.通知下一个线程 

主要注意的是首先`synchronized`对象，在`synchronized`块内部才能使用`wait`和`notify`

其实和第一种方式是一样的，只是实现语法不同。

# 三. 其他

`CountDownLatch`，`Semphore`等都可以实现该需求，这里就不一一实现了，

关键是要理解如何分配共享资源、如何通知共享资源。