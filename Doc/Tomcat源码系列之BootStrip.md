Tomcat入口程序位于org.apache.catalina.startup.BootStrip

# static静态块
> 设置catalina.home和catalina.base两个环境变量

查找环境变量路径的逻辑如下
1. 如果当前catalina.home环境变量有值，则用当前值初始化变量
2. 否则，如果当前目录存在bootstrap.jar文件，则用当前目录的上级目录初始化
3. 否则，采用当前目录

# init
1. 初始化classLoader。

> commonLoader、catalinaLoader、sharedLoader，其中commonLoader是其他两个加载类的父类。
*此处的父子关系是逻辑上的父子关系，并不是类继承关系*

> commonLoader生成时候首先读取conf/catalina.properties文件中commLoader.loader属性：默认为${catalina.base}/lib、${catalina.base}/lib/*.jar、${catalina.home}/lib、${catalina.home}/lib/*.jar

2. Thread.currentThread().setContextClassLoader(catalinaLoader)
3. SecurityClassLoad.securityClassLoad(catalinaLoader);
4. 设置org.apache.catalina.startup.Catalina的setParentClassLoader为Bootstrip
5. catalinaDaemon为Catalina