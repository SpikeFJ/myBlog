## mysql安装问题
1. 下载`mysql-8.0.17-winx64`
2. 解压到`D:\soft\mysql-8.0.17-winx64`
3. `cd bin`,定位到`bin`目录
4. 执行mysql.exe报错


1. 执行`mysqld -- install `，安装mysql服务
2. 尝试启动服务，失败
3. 在`D:\soft\mysql-8.0.17-winx64`下新建`data`文件夹
4. 执行`mysqld -remove MySQL`后再次重新安装mysql服务：`mysqld --install mysql`


5. 报错后可以在`data`文件夹下找到`CNHQ-19043913N.err`错误文件
6. google文件错误信息

解决方法：
1. 手动删掉自己创建的data文件夹
2. 然后再管理员cmd下进入 bin 目录，移除自己的mysqld服务
3. 在cmd的bin目录执行 mysqld --initialize-insecure
程序会在动MySQL文件夹下创建data文件夹以及对应的文件
4. bin目录下执行，mysqld --install ，安装mysqld服务
5. 在bin目录下运行net start mysql ,启动mysql服务。