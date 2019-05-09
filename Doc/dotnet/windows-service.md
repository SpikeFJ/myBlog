项目中发现很多类似于自动任务、控制台窗口运行的程序需要稳定的在后台运行，于是查找了C#中部署windows服务相关内容，内容如下：

* [程序开发](#1.程序开发)
* [服务安装](#2.服务安装)

# 1.程序开发
## 1.1 vs中新建windows服务项目
<p>
vs会自动生成program.cs和Service1.cs文件，<br>
Service1继承ServiceBase类，可以重写OnStart等方法<br>
Program中main方以如下方法调用服务基类：

```
ServiceBase[] ServicesToRun;
ServicesToRun = new ServiceBase[]
{
    new Service1()
};
ServiceBase.Run(ServicesToRun);
```
## 1.2 根据Service1生成安装服务
双击Service1.cs出现【设计】界面，<br>
在设计界面右击，“添加安装程序”，会自动生成ProjectInstaller.cs，<br>
ProjectInstaller中含有两个对象(ServiceProcessInstaller、ServiceInstaller)，<br>
可以设置ServiceInstaller对象的属性，如ServiceName,StartType来控制服务的名称、启动方式等。



# 2.服务安装
服务安装有多种方式，常用的有命令行、代码方式。

## 2.1 命令行安装
```
安装：installutil WindowsService1.exe

卸载：installutil /u WindowsService1.exe
```
实际项目中，可以将命令行组织成批处理文件：<br>
<p>
安装并启动.bat

```
"%cd%\InstallUtil.exe"  "%cd%\CocoWatcher.exe"
net start "CocoWatcher"
pause
```

卸载并停止.bat
```
net stop "CocoWatcher"
"%cd%\InstallUtil.exe" "%cd%\CocoWatcher.exe"  -u
taskkill /f /im CocoWatcher.exe
pause
```


## 2.2 代码安装(InstallUtil)
该方法本质还是通过命令行安装，只不过是通过Process启动。<br>
提供了一种通过程序安装的途径。
1. 找到InstallUtil.exe。<br>
    1.1 通过注册表获取
    ```
    private string GetInstallutilPath()
        {
            Microsoft.Win32.RegistryKey rkServices = Microsoft.Win32.Registry.LocalMachine.OpenSubKey(@"SOFTWARE\Microsoft\.NETFramework");
            string strFrameworkPath = AMS.Monitoring.Function.ToString(rkServices.GetValue("InstallRoot"));
            if (strFrameworkPath.Length > 0)
            {
                if (System.IO.Directory.Exists(strFrameworkPath))
                {
                    foreach (string strDirectory in System.IO.Directory.GetDirectories(strFrameworkPath))
                    {
                        string[] strFolderNames = strDirectory.Split('\\');
                        if (strFolderNames.Length > 0)
                        {
                            if (strFolderNames[strFolderNames.Length - 1].StartsWith("v4."))
                            {
                                string strInstallUtilPath = strDirectory + "\\InstallUtil.exe";
                                if (System.IO.File.Exists(strInstallUtilPath))
                                {
                                    return strInstallUtilPath;
                                }
                            }
                        }
                    }
                }
            }
            return null;
        }
    ```
    1.2 通过net自带API获取
    ```
    public static string InstallUtilPath = System.Runtime.InteropServices.RuntimeEnvironment.GetRuntimeDirectory(); 
    proc.StartInfo.FileName = Path.Combine(InstallUtilPath, "installutil.exe");

    ```

2. 通过Process类启动InstallUtil.exe
   ```
    System.Diagnostics.ProcessStartInfo psi = new System.Diagnostics.ProcessStartInfo();
    psi.FileName = strInstallutilPath;
    psi.Arguments = this.GetType().Assembly.CodeBase.Substring(8);
    psi.WindowStyle = System.Diagnostics.ProcessWindowStyle.Normal;
    System.Diagnostics.Process pro = System.Diagnostics.Process.Start(psi);
    do
    {
        System.Threading.Thread.Sleep(1);
    }
    while (pro.HasExited == false);

   ```
## 2.3 代码安装
  此方法参考博客园「焰尾迭」，具体逻辑
  1. 加载程序集、反射得到ServiceInstaller的诸多属性，由此就知道了服务名称、路径、描述等信息
  2. 安装服务
  ```
TransactedInstaller transactedInstaller = new TransactedInstaller();
AssemblyInstaller assemblyInstaller = new AssemblyInstaller(serviceFileName, cmdline);
transactedInstaller.Installers.Add(assemblyInstaller);
transactedInstaller.Install(new System.Collections.Hashtable());
  ```
ps:实际上installutil.exe也是利用transactedinstaller、assemblyinstaller等类实现的<br>

  3. 运行服务
  ```
 ServiceController service = new ServiceController(serviceName);
if (service.Status != ServiceControllerStatus.Running && service.Status != ServiceControllerStatus.StartPending)
{
    service.Start();
    txtTip.Text = "服务运行成功！";
}
else
{
    txtTip.Text = "服务正在运行！";
}
  ```
  4. 停止服务
  ```
ServiceController service = new ServiceController(serviceName);
if (service.Status == ServiceControllerStatus.Running)
{
    service.Stop();
    txtTip.Text = "服务停止成功！";
}
else
{
    txtTip.Text = "服务已经停止！";
}
  ```
参考资料：<br>
1. [更上层楼：动态安装你的windows服务](http://www.cnblogs.com/jeffwongishandsome/archive/2011/03/12/1981822.html) <br>
2. [用c#实现通用守护进程](http://www.cnblogs.com/tianzhiliang/archive/2011/02/12/1952221.html)<br>
3. [使用工具安装，运行，停止，卸载Window服务](http://www.cnblogs.com/yanweidie/p/3542670.html)