今天在使用`maven`下载私服`jar`包时遇到了一些问题，顺便了解了下`maven`的一些基础配置，记录下：

首先需要了解的是`maven`的基本操作流程

`maven`首先从本地仓库获取`jar`包，如果没有则从私服（如果有的话）获取，最后从中央仓库获取

# 1. 本地仓库

本地仓库默认地址为：`${user.home}/.m2/repository`

可以通过`settings.xml`修改本地仓库地址：

```xml
 <localRepository>D:/.m2/repository</localRepository>
```

也可以在运行时指定本地仓库地址：

```xml
mvn clean install -Dmaven.repo.local=/home/juven/myrepo/
```

我们常说的安装，即`install`，就是将本地项目打包复制到本地仓库中去

# 2. 中央仓库

这里先越过私服仓库，先讲下中央仓库的概念，默认中央仓库在哪里配置呢？

找到`maven`安装位置l，定位到`D:\soft\apache-maven-3.2.2\lib\maven-model-builder-3.2.2.jar`，解压缩后可以找到`\org\apache\maven\model\pom-4.0.0.xml`

该`pom`文件是所有`maven`项目`pom`文件的父`pom`，即所有`maven`项目都继承该配置，在该`pom`文件中有如下配置：

```xml
<repositories>
    <repository>
      <id>central</id>
      <name>Central Repository</name>
      <url>https://repo.maven.apache.org/maven2</url>
      <layout>default</layout>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>

```

可以看到，这里指明了默认的中央仓库地址：`https://repo.maven.apache.org/maven2`

# 3. 私服仓库

很多时候，企业会在局域网内部署一套`maven`仓库，即能提升下载速度，又能保存一些不便于公开的`jar`包，配置如下：

```xml
<repositories>
    <repository>
      <id>maven-net-cn</id>
      <name>Maven China Mirror</name>
      <url>192.168.1.111/groups/public/</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>


```

`repository`表明一个仓库地址，可以同时配置多个仓库；

`repository`有`ID`，`name`及`url`，

`<releases><enabled>true</enabled></releases>`表示`maven`可以从该仓库下载`releases`版本`jar`包

`<snapshots><enabled>false</enabled></snapshots>`表示`maven`不要从该仓库下载`snapshots`版本`jar`包

# 4. 插件仓库

一般和`repositories`一起出现的元素，还有`pluginRepositories`，

要了解`pluginRepositories`，首先要知道`maven`本身是由`java`语言开发，并且采用了插件式的开发模式，

`maven`本质上是一个插件框架，它的核心并不执行任何具体的构建任务，所有这些任务都交给插件来完成

在`maven`执行的不同阶段都预留了相应的接口以便开发者扩展

常用的插件有`maven-assembly-plugin`,`maven-surefire-plugin`等。

那么看下`pluginRepositories`的使用方法如下：

```xml
  <pluginRepositories>
    <pluginRepository>
      <id>maven-net-cn</id>
      <name>Maven China Mirror</name>
      <url>http://maven.net.cn/content/groups/public/</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>    
    </pluginRepository>
  </pluginRepositories>
```

可以看到基本和`repository`类似，不同的是`pluginRepositories`下载的是插件的`jar`包，`repository`下载的是具体项目的`jar`包

# 5. `setting`文件

## 1. 配置相同参数

以上配置都是针对单个项目的`pom`文件，如果有多个项目，一个个配置实在繁琐，所幸有`setting.xml`全局配置，使用方法如下：

```xml

<settings>
  ...
  <profiles>
    <profile>
      <id>dev</id>
      <!-- repositories and pluginRepositories here-->
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>dev</activeProfile>
  </activeProfiles>
  ...
</settings>
```

定义一个`id`为`dev`的配置节点`profile`，`profile`节点中配置相关参数；

`activeProfiles`中指定哪一个`profile`节点生效。



## 2. 镜像

```xml

<settings>
...
  <mirrors>
    <mirror>
      <id>maven-net-cn</id>
      <name>Maven China Mirror</name>
      <url>http://maven.net.cn/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
...
</settings>
```

