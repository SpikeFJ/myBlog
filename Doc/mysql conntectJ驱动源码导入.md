# 简介
    
jdbc是Java平台提供的访问数据库的同意API，数据库厂商提供该API的实现(jar包)，由此带来的好处就是应用程序只是针对接口编程，数据库切换只需要引用不同的jar包，不需要更改具体程序。


# 使用场景
```
    Class.forName("com.mysql.jdbc.Driver");
    Connection conn =DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root", "123456");

    Statement stmt = conn.createStatement();
    stmt.executeUpdate("DROP TABLE IF EXISTS my_table");
    stmt.executeUpdate("CREATE TABLE IF NOT EXISTS my_table(name varchar(20))");
    stmt.executeUpdate("INSERT INTO my_table(name) VALUES('zhh')");

    ResultSet rs = stmt.executeQuery("SELECT name FROM my_table");
    rs.next();
    System.out.println(rs.getString(1));

    stmt.close();
    conn.close();
```

# 安装部署
参考[官方教程](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-installing-source.html)

    1.下载源码
    2.下载jdk5
    3.下载Hibernatejar
    4.配置build.xml
    

# 分析

1. DriveManage静态块加载驱动,知识点
    * AccessController.doPrivileged
    * ServiceLoader，此处设计classloader加载文件
    参考：[关于getSystemResource, getResource 的总结](http://www.cnblogs.com/drwong/p/5389631.html)
    * CopyOnWriteArrayList

    实现IDriver的类都需要调用DriveManage.regist注册资金

    
