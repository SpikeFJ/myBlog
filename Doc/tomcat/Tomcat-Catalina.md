BootStrip作为一个启动类，实际上大部分方法如load，start执行的却是Catalina的方法。

直接启动Catalina实例就可以了，为什么还需要Bootstrip类呢，我的理解：Catalina作为核心环境成员应该与外界解耦，bootstrip充当了外界和Catalina的联络员。

在分析之前，所有要理清下Tomcat的整体框架。

Tomcat本质是一个Web容器，所以Container接口作为所有容器的基类，其中StandardServer作为最外层的容器类，StandardServer包含多个StandardService，每一个Service都由多个Connector和一个StandardEngine组成，所以整体结构如下

对容器的基础方法的调用(如初始化、init、start等方法)不仅仅会执行自身的逻辑，还会递归调用下级容器相应的方法。

如Server调用init，会依次执行Service、conncetor、Enginer、Host、Content、Wrap等类的init方法。

# 一.加载(load)
由于对JMS相关不是很了解，所以如initNaming等方法暂时忽略，关注整体流程

1. initDirs
2. initNaming
3. createStartDigester并解析(
*此处会对调用StandardEngine的初始化和addChild方法* )
4. StandardServer设置CatalinaHome、CatalinaBase
5. initStreams
6. StandardServer执行init()


**此处重点关注`createStartDigester`方法，该方法加载解析`conf/server.xml`文件，根据相应的规则实例化Tomcat各个组件，并将各组件组合在一起。**


# 二.启动(start)
1. 执行StandardServer的start方法，如果失败执行destroy方法

2. 添加jvm钩子。
CatalinaShutdownHook是一个线程类，run方法中调用Catalina的stop方法
```java
shutdownHook = new CatalinaShutdownHook();
Runtime.getRuntime().addShutdownHook(shutdownHook)
```
```java
Runtime.getRuntime().removeShutdownHook(shutdownHook);
 Server s = getServer();
 s.stop();
 s.destroy();
```

3. 阻塞等待。await调用StandardServer的await方法 
    ```java
    //await变量在bootStrip中置为true
    if (await) {
        await();
        stop();
    }
    ```

# TODO
* StringManger分析
* JavaXml、setSecurityProtection
