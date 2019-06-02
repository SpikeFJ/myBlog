上篇分析到HostConfig的入口方法：lifeCycleEvent
```java
setCopyXML(((StandardHost) host).isCopyXML());//是否发布xml文件的标识，默认为true  
setDeployXML(((StandardHost) host).isDeployXML());;//是否动态部署标识，默认为true  
setUnpackWARs(((StandardHost) host).isUnpackWARs());//是否要将war文件解压缩，默认为true
setContextClass(((StandardHost) host).getContextClass());
```

# beforeStart

检查webapps和conf/localhost文件夹是否存在

# start

* 检查是否文件夹：C:\workspace\Tomcat9\webapps；判断为否则DeployOnStartup，AutoDeploy设置为false
* DeployOnStartup=true，则调用deployApps
```java
//C:\workspace\Tomcat9\webapps
File appBase = host.getAppBaseFile();
//C:\workspace\Tomcat9\conf\Catalina\localhost
File configBase = host.getConfigBaseFile();
String[] filteredAppPaths = filterAppPaths(appBase.list());
// Deploy XML descriptors from configBase
deployDescriptors(configBase, configBase.list());//通过描述符发布应用  
// Deploy WARs
deployWARs(appBase, filteredAppPaths);//发布war文件的应用  
// Deploy expanded folders
deployDirectories(appBase, filteredAppPaths);//发布目录型的应用 
```
* **在deployDirectory部署阶段会反射调用StandardContext，并且在往StandardHost中addChild时会触发Context的init，start等操作。**

## deployDescriptors分析：
* 查找C:\workspace\Tomcat9\conf\Catalina\localhost下所有xml文件组织成ContextName
* 开线程调用digester解析该文件，生成StandardContext
* 初始化StandardContext，设置相关属性及监听器
* 当该文件的DocBase属性不为null时

## deployWARs分析
* 忽略所有META-INF、WEB-INF命名的文件夹
* 查找C:\workspace\Tomcat9下所有.war包