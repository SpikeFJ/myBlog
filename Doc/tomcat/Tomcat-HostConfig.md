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
//根目录下/weapps   eg:C:\workspace\Tomcat9\webapps
File appBase = host.getAppBaseFile();
//根目录下/conf/Catalina/localhost eg:C:\workspace\Tomcat9\conf\Catalina\localhost
File configBase = host.getConfigBaseFile();
String[] filteredAppPaths = filterAppPaths(appBase.list());
// Deploy XML descriptors from configBase
deployDescriptors(configBase, configBase.list());//通过描述符发布应用  
// Deploy WARs
deployWARs(appBase, filteredAppPaths);//发布war文件的应用  
// Deploy expanded folders
deployDirectories(appBase, filteredAppPaths);//发布目录型的应用 
```
**在deployDirectory部署阶段会反射调用StandardContext，并且在往StandardHost中addChild时会触发Context的init，start等操作。**


----
以下章节重点分析web应用程序(context)的加载：


# deployDescriptors分析：
主要步骤
1. 查找C:\workspace\Tomcat9\conf\Catalina\localhost下所有xml文件组织成ContextName
2. 开线程调用hostconfig的deployDescriptor方法，该方法会调用digester解析该文件，生成StandardContext
3. 初始化StandardContext，设置相关属性及监听器
4. 当该文件的DocBase属性不为null时

```java
/**
    configBase:C:\workspace\Tomcat9\conf\Catalina\localhost
    files:xxxx.xml
**/
protected void deployDescriptors(File configBase, String[] files) {

    if (files == null)
        return;

    ExecutorService es = host.getStartStopExecutor();
    List<Future<?>> results = new ArrayList<>();

    for (int i = 0; i < files.length; i++) {
        File contextXml = new File(configBase, files[i]);

        //判断是否后缀为.xml
        if (files[i].toLowerCase(Locale.ENGLISH).endsWith(".xml")) {
            ContextName cn = new ContextName(files[i], true);

            if (isServiced(cn.getName()) || deploymentExists(cn.getName()))
                continue;
            //开线程启动DeployDescriptor对象
            results.add(es.submit(new DeployDescriptor(this, cn, contextXml)));
        }
    }

    for (Future<?> result : results) {
        try {
            result.get();
        } catch (Exception e) {
            log.error(sm.getString(
                    "hostConfig.deployDescriptor.threaded.error"), e);
        }
    }
}


```
## 1. ContextName分析

具体代码如下：
```java

public ContextName(String name, boolean stripFileExtension) {

        String tmp1 = name;
        //如果/xx.xml，则转换为xx.xml
        if (tmp1.startsWith("/")) {
            tmp1 = tmp1.substring(1);
        }

        // 把所有的/替换为#，如xx/yy/zz.xml-->xx#yy#zz.xml
        tmp1 = tmp1.replaceAll("/", FWD_SLASH_REPLACEMENT);

        //1.##xx.xml-->Root##xx.xml
        //2./.xml----->Root.xml 
        if (tmp1.startsWith(VERSION_MARKER) || "".equals(tmp1)) {
            tmp1 = ROOT_NAME + tmp1;
        }

        // stripFileExtension=true时，后缀名截断
        if (stripFileExtension &&
                (tmp1.toLowerCase(Locale.ENGLISH).endsWith(".war") ||
                        tmp1.toLowerCase(Locale.ENGLISH).endsWith(".xml"))) {
            tmp1 = tmp1.substring(0, tmp1.length() -4);
        }

        baseName = tmp1;

        //获取版本信息
        String tmp2;
        int versionIndex = baseName.indexOf(VERSION_MARKER);
        if (versionIndex > -1) {
            version = baseName.substring(versionIndex + 2);
            tmp2 = baseName.substring(0, versionIndex);
        } else {
            version = "";
            tmp2 = baseName;
        }

        if (ROOT_NAME.equals(tmp2)) {
            path = "";
        } else {
            path = "/" + tmp2.replaceAll(FWD_SLASH_REPLACEMENT, "/");
        }

        //name：path+#+版本号
        if (versionIndex > -1) {
            this.name = path + VERSION_MARKER + version;
        } else {
            this.name = path;
        }
    }
```

ContextName的构造函数会对传入的xx.xml进行分析截取，具体步骤如下：
1. 如果xml文件以为`/`开头，则去除`/`
2. 把所有的`/`都替换成`#`
> 步骤1.2在windows上其实是多余的。因为新建文件时会报以下错误：“文件名不能包含下列任何字符：\/:*?"<>|”
3. 如果文件以`##`开头，则替换为`ROOT`+文件名;（即如果含有版本号但没有名称，则认为是root的分支版本）
   如果文件为`空`,则替换为`Root.xml`
4. 根据需要截断后缀名
5. 经过以上4个步骤生成的xml文件名赋值给`baseName`(baseName是含有版本号的字符串)
6. 如果`baseName`中含有`##`,则`##`之后的字符串作为版本号(`version`)，之前的的字符串作为`tmp2`
7. 如果`tmp2`等于`ROOT`，则`path=""`,否则path= `/`+tmp2将所有`#`替换为`/`后的字符串
8. `version`不为空时：`name`等于`path`+`##`+`version`；否则`name`等于`path`



## 2. deployDescriptor分析
```java
 protected void deployDescriptor(ContextName cn, File contextXml) {

        DeployedApplication deployedApp =
                new DeployedApplication(cn.getName(), true);

        long startTime = 0;
        // Assume this is a configuration descriptor and deploy it
        if(log.isInfoEnabled()) {
           startTime = System.currentTimeMillis();
           log.info(sm.getString("hostConfig.deployDescriptor",
                    contextXml.getAbsolutePath()));
        }

        Context context = null;
        boolean isExternalWar = false;
        boolean isExternal = false;
        File expandedDocBase = null;

       
        try (FileInputStream fis = new FileInputStream(contextXml)) {
             //采用digester解析xml文件
            synchronized (digesterLock) {
                try {
                    context = (Context) digester.parse(fis);
                } catch (Exception e) {
                    log.error(sm.getString(
                            "hostConfig.deployDescriptor.error",
                            contextXml.getAbsolutePath()), e);
                } finally {
                    digester.reset();
                    if (context == null) {
                        context = new FailedContext();
                    }
                }
            }
            //为context对象添加contextConfig监听器
            Class<?> clazz = Class.forName(host.getConfigClass());
            LifecycleListener listener =
                (LifecycleListener) clazz.newInstance();
            context.addLifecycleListener(listener);
            //根据xml配置context相关属性
            context.setConfigFile(contextXml.toURI().toURL());
            context.setName(cn.getName());
            context.setPath(cn.getPath());
            context.setWebappVersion(cn.getVersion());
            // 如果xml中docBase属性不为空
            if (context.getDocBase() != null) {
                File docBase = new File(context.getDocBase());
                //如果是相对地址，则采用当前主机路径
                if (!docBase.isAbsolute()) {
                    docBase = new File(host.getAppBaseFile(), context.getDocBase());
                }
                // 如果<Context docBase="D:/example />中的路径不是在host路径中
                if (!docBase.getCanonicalPath().startsWith(
                        host.getAppBaseFile().getAbsolutePath() + File.separator)) {
                    //标志为外部项目
                    isExternal = true;
                    
                    deployedApp.redeployResources.put(
                            contextXml.getAbsolutePath(),
                            Long.valueOf(contextXml.lastModified()));
                    deployedApp.redeployResources.put(docBase.getAbsolutePath(),
                            Long.valueOf(docBase.lastModified()));
                    if (docBase.getAbsolutePath().toLowerCase(Locale.ENGLISH).endsWith(".war")) {
                        isExternalWar = true;
                    }
                } else {
                    log.warn(sm.getString("hostConfig.deployDescriptor.localDocBaseSpecified",
                             docBase));
                    // Ignore specified docBase
                    context.setDocBase(null);
                }
            }
            //将web项目添加到host中，此处会触发context的init-->start；而init、start之前又会触发conextConfig的事件响应
            host.addChild(context);
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            log.error(sm.getString("hostConfig.deployDescriptor.error",
                                   contextXml.getAbsolutePath()), t);
        } finally {
            // Get paths for WAR and expanded WAR in appBase

            // default to appBase dir + name
            expandedDocBase = new File(host.getAppBaseFile(), cn.getBaseName());
            if (context.getDocBase() != null
                    && !context.getDocBase().toLowerCase(Locale.ENGLISH).endsWith(".war")) {
                // first assume docBase is absolute
                expandedDocBase = new File(context.getDocBase());
                if (!expandedDocBase.isAbsolute()) {
                    // if docBase specified and relative, it must be relative to appBase
                    expandedDocBase = new File(host.getAppBaseFile(), context.getDocBase());
                }
            }

            boolean unpackWAR = unpackWARs;
            if (unpackWAR && context instanceof StandardContext) {
                unpackWAR = ((StandardContext) context).getUnpackWAR();
            }

            // Add the eventual unpacked WAR and all the resources which will be
            // watched inside it
            if (isExternalWar) {
                if (unpackWAR) {
                    deployedApp.redeployResources.put(expandedDocBase.getAbsolutePath(),
                            Long.valueOf(expandedDocBase.lastModified()));
                    addWatchedResources(deployedApp, expandedDocBase.getAbsolutePath(), context);
                } else {
                    addWatchedResources(deployedApp, null, context);
                }
            } else {
                // Find an existing matching war and expanded folder
                if (!isExternal) {
                    File warDocBase = new File(expandedDocBase.getAbsolutePath() + ".war");
                    if (warDocBase.exists()) {
                        deployedApp.redeployResources.put(warDocBase.getAbsolutePath(),
                                Long.valueOf(warDocBase.lastModified()));
                    } else {
                        // Trigger a redeploy if a WAR is added
                        deployedApp.redeployResources.put(
                                warDocBase.getAbsolutePath(),
                                Long.valueOf(0));
                    }
                }
                if (unpackWAR) {
                    deployedApp.redeployResources.put(expandedDocBase.getAbsolutePath(),
                            Long.valueOf(expandedDocBase.lastModified()));
                    addWatchedResources(deployedApp,
                            expandedDocBase.getAbsolutePath(), context);
                } else {
                    addWatchedResources(deployedApp, null, context);
                }
                if (!isExternal) {
                    // For external docBases, the context.xml will have been
                    // added above.
                    deployedApp.redeployResources.put(
                            contextXml.getAbsolutePath(),
                            Long.valueOf(contextXml.lastModified()));
                }
            }
            // Add the global redeploy resources (which are never deleted) at
            // the end so they don't interfere with the deletion process
            addGlobalRedeployResources(deployedApp);
        }

        //加入到deployed，表示已加载完成的应用程序。
        if (host.findChild(context.getName()) != null) {
            deployed.put(context.getName(), deployedApp);
        }

        if (log.isInfoEnabled()) {
            log.info(sm.getString("hostConfig.deployDescriptor.finished",
                contextXml.getAbsolutePath(), Long.valueOf(System.currentTimeMillis() - startTime)));
        }
    }
```

## 3. deployWARs分析和deployDirectories就不分析了，基础代码差不多。
* 忽略所有META-INF、WEB-INF命名的文件夹
* 查找C:\workspace\Tomcat9下所有.war包
* 其他和deployDescriptor相同

# 总结

host主要聚焦于context的加载，初始化、启动，下面继续分析Context对象