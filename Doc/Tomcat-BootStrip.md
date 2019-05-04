Tomcat入口程序位于org.apache.catalina.startup.BootStrip

# static静态块
设置catalina.home和catalina.base两个环境变量。

查找环境变量路径的逻辑如下

1. 如果当前catalina.home环境变量有值，则用当前值初始化变量
2. 否则，如果当前目录存在bootstrap.jar文件，则用当前目录的上级目录初始化
3. 否则，采用当前目录

*catalina.home和catalina.base分别代表安装目录和工作目录，可以通过设置catalina.base来启用Tomcat多实例。*

```
ps:
代码中有很多通过`System.getProperty` 获取自定义变量，getProperty可以有以下两种方式设置而来：
1.通过-Dkey=value方式在启动java时配置
2.修改catalina.bat文件，Set Java_OPTS=-Dkey=value
```

# 初始化
1. 初始化classLoader。Tomcat提供3种类加载器，分别为 commonLoader、catalinaLoader、sharedLoader,其中commonLoader是其他两种加载器的父加载器。catalinaLoader作为Tomcat自身运行所需要的类加载器，sharedLoader作为所有Web项目运行所需要的类加载器。

    *此处的父子关系是逻辑上的父子关系，并不是类继承关系*

     类加载器路径在conf/catalina.properties指定，分别对应common.loader，server.loader，shared.loader属性。

2. Thread.currentThread().setContextClassLoader(catalinaLoader)
3. SecurityClassLoad.securityClassLoad(catalinaLoader);
4. 反射调用org.apache.catalina.startup.Catalina的setParentClassLoader为sharedLoader
5. catalinaDaemon为Catalina(Catalina构造函数会调用setSecurityProtection进行安全等相关设置)

# 开始

```java
    daemon.setAwait(true); //调用Catalina.SetAwait
    daemon.load(args);//Catalia.load-->StandadServer初始化
    daemon.start();
```
<p>
此处会设置Catalina的isAwait=true,保证Catalina在start之后会阻塞等待，而不是立刻跳出方法。
</p>

参见下一节Catalina解析


### TODO
* Jdk日志库、Tomcat日志库
* 类加载机制