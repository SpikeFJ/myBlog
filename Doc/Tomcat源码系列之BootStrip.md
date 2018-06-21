Tomcat入口程序位于org.apache.catalina.startup.BootStrip

# static静态块
> 设置catalina.home和catalina.base两个环境变量

查找环境变量路径的逻辑如下
1. 如果当前catalina.home环境变量有值，则用当前值初始化变量
2. 否则，如果当前目录存在bootstrap.jar文件，则用当前目录的上级目录初始化
3. 否则，采用当前目录

# init
1. 初始化classLoader。Tomcat提供3种类加载器，分别为 commonLoader、catalinaLoader、sharedLoader,其中commonLoader是其他两种加载器的父加载器。

    *此处的父子关系是逻辑上的父子关系，并不是类继承关系*

2. 三种类加载器对应的路径在conf/catalina.properties指定，分别对应common.loader，server.loader，shared.loader属性。

2. Thread.currentThread().setContextClassLoader(catalinaLoader)
3. SecurityClassLoad.securityClassLoad(catalinaLoader);
4. 设置org.apache.catalina.startup.Catalina的setParentClassLoader为Bootstrip
5. catalinaDaemon为Catalina