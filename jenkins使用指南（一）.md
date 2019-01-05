## 运行环境配置
* windows10操作系统
* 安装java
* 安装maven
* 下载Jenkins.war（D:\Jenkins.war）

## 新建项目 
新建一个maven项目，执行一条语句helloworld，作为我们需要构建的源头。

## 启动Jenkins
```
cd D:
java -jar jenkins.war --httpPort=8080
```
打开浏览器，输入http://localhost:8080即可

参考资料：
- [jenkins指南](https://blog.csdn.net/li_yan_fei/article/details/79077051)