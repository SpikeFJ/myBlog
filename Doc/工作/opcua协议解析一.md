代码实现来自于https://github.com/OPCFoundation/UA-.NETStandard

该解决方案包括了大量实际使用样例，我们先关注 `Empty`文件夹下的 `Empty Server`项目

# 一. 开始

首先从`Program`入手：

```c#
ApplicationInstance application = new ApplicationInstance();
application.ApplicationType   = ApplicationType.Server;
application.ConfigSectionName = "Quickstarts.EmptyServer";

try
{
    // 加载框架配置
    application.LoadApplicationConfiguration(false).Wait();

    //检查框架证书
    application.CheckApplicationInstanceCertificate(false, 0).Wait();

    //启动服务
    application.Start(new EmptyServer()).Wait();

    //运行
    Application.Run(new Opc.Ua.Server.Controls.ServerForm(application));
}
catch (Exception e)
{
    ExceptionDlg.Show(application.ApplicationName, e);
    return;
}
```

## 1.1.  加载框架配置

```C#
 public async Task<ApplicationConfiguration> LoadApplicationConfiguration(bool silent)
 {
     //1.找到文件名称为Quickstarts.EmptyServer.Config.xml的实际路径
     string filePath = ApplicationConfiguration.GetFilePathFromAppConfig(ConfigSectionName);
     //
     ApplicationConfiguration configuration = await LoadAppConfig(silent, filePath, ApplicationType, ConfigurationType, true);

     if (configuration == null)
     {
         throw ServiceResultException.Create(StatusCodes.BadConfigurationError, "Could not load configuration file.");
     }

     m_applicationConfiguration = FixupAppConfig(configuration);

     return m_applicationConfiguration;
 }

//最终加载配置的方法
 public static async Task<ApplicationConfiguration> Load(FileInfo file, ApplicationType applicationType, Type systemType, bool applyTraceSettings)
 {
     ApplicationConfiguration configuration = null;

     if (systemType == null)
     {
         systemType = typeof(ApplicationConfiguration);
     }

     FileStream stream = new FileStream(file.FullName, FileMode.Open, FileAccess.Read);

     try
     {
         //将配置文件实例化成ApplicationConfiguration对象
         DataContractSerializer serializer = new DataContractSerializer(systemType);
         configuration = (ApplicationConfiguration)serializer.ReadObject(stream);
     }
     catch (Exception e)
     {
         throw ServiceResultException.Create(
             StatusCodes.BadConfigurationError,
             e,
             "Configuration file could not be loaded: {0}\r\nError is: {1}",
             file.FullName,
             e.Message);
     }
     finally
     {
         stream.Dispose();
     }

     if (configuration != null)
     {
         // should not be here but need to preserve old behavior.
         if (applyTraceSettings && configuration.TraceConfiguration != null)
         {
             //设置日志相关配置(路径、异常等级等)
             configuration.TraceConfiguration.ApplySettings();
         }

         //校验配置文件信息，主要是证书的验证
         await configuration.Validate(applicationType);

         configuration.m_sourceFilePath = file.FullName;
     }

     return configuration;
 }
```

## 1.2. 检查证书信息

这里不做过多介绍

## 1.3. 启动服务

```C#
//ApplicationInstance.cs
public async Task Start(ServerBase server)
{
    m_server = server;

    //1.确保配置文件被加载
    if (m_applicationConfiguration == null)
    {
        await LoadApplicationConfiguration(false);
    }

    //2.确保证书被加载
    if (m_applicationConfiguration.CertificateValidator != null)
    {
        m_applicationConfiguration.CertificateValidator.CertificateValidation += CertificateValidator_CertificateValidation;
    }

    //启动server
    server.Start(m_applicationConfiguration);
}

```

# 二. Server启动

进入到`server`，下面是`start`的执行步骤：

```c#
public void Start(ApplicationConfiguration configuration)
{
    if (configuration == null) throw new ArgumentNullException("configuration");

    //1.开启前的准备
    OnServerStarting(configuration);
	//2.初始化RequestQueue
    InitializeRequestQueue(configuration);
	//3.初始化服务器 能力？
    ServerCapabilities = configuration.ServerConfiguration.ServerCapabilities;
	//4.初始化baseAddress
    InitializeBaseAddresses(configuration);
	//5.初始化hosts
    ApplicationDescription serverDescription = null;
    EndpointDescriptionCollection endpoints = null;

    IList<Task> hosts = InitializeServiceHosts(
        configuration,
        out serverDescription,
        out endpoints);

    //6.保存发现的信息endpoints
    ServerDescription = serverDescription;
    m_endpoints = new ReadOnlyList<EndpointDescription>(endpoints);

    //7.开启框架
    StartApplication(configuration);

    //8.打开hosts.
    lock (m_hosts)
    {
        foreach (Task serviceHost in hosts)
        {
            m_hosts.Add(serviceHost);
        }
    }
}


```

### 2.1. `OnServerStarting`

`StandServer`对该方法进行了重写：

```c#
protected override void OnServerStarting(ApplicationConfiguration configuration)
{
    lock (m_lock)
    {
        base.OnServerStarting(configuration);
        m_minNonceLength = configuration.SecurityConfiguration.NonceLength;
        m_useRegisterServer2 = true;
    }
}
```

几乎没什么操作，还是看基类`ServerBase`实现：

```C#
protected virtual void OnServerStarting(ApplicationConfiguration configuration)
{
    //加载属性
    Configuration = configuration;
    ServerProperties = LoadServerProperties();

    // 确保至少一个安全策略文件存在
    if (configuration.ServerConfiguration != null)
    {
        if (configuration.ServerConfiguration.SecurityPolicies.Count == 0)
        {
            configuration.ServerConfiguration.SecurityPolicies.Add(new ServerSecurityPolicy());
        }

        //确保至少一个UserToken策略存在
        if (configuration.ServerConfiguration.UserTokenPolicies.Count == 0)
        {
            UserTokenPolicy userTokenPolicy = new UserTokenPolicy();

            userTokenPolicy.TokenType = UserTokenType.Anonymous;
            userTokenPolicy.PolicyId = userTokenPolicy.TokenType.ToString();

            configuration.ServerConfiguration.UserTokenPolicies.Add(userTokenPolicy);
        }
    }

    // 加载证书实例，略过.
    if (configuration.SecurityConfiguration.ApplicationCertificate != null)
    {
        InstanceCertificate = configuration.SecurityConfiguration.ApplicationCertificate.Find(true).Result;
    }

    if (InstanceCertificate == null)
    {
        throw new ServiceResultException(
            StatusCodes.BadConfigurationError,
            "Server does not have an instance certificate assigned.");
    }

    if (!InstanceCertificate.HasPrivateKey)
    {
        throw new ServiceResultException(
            StatusCodes.BadConfigurationError,
            "Server does not have access to the private key for the instance certificate.");
    }

    //加载证书链，略过
    InstanceCertificateChain = new X509Certificate2Collection(InstanceCertificate);
    List<CertificateIdentifier> issuers = new List<CertificateIdentifier>();
    configuration.CertificateValidator.GetIssuers(InstanceCertificateChain, issuers).Wait();

    for (int i = 0; i < issuers.Count; i++)
    {
        InstanceCertificateChain.Add(issuers[i].Certificate);
    }

    //通过configuration创建ServiceMessageContext对象
    MessageContext = configuration.CreateMessageContext();

    // assign a unique identifier if none specified.
    if (String.IsNullOrEmpty(configuration.ApplicationUri))
    {
        configuration.ApplicationUri = Utils.GetApplicationUriFromCertificate(InstanceCertificate);

        if (String.IsNullOrEmpty(configuration.ApplicationUri))
        {
            configuration.ApplicationUri = Utils.Format(
                "http://{0}/{1}/{2}",
                Utils.GetHostName(),
                configuration.ApplicationName,
                Guid.NewGuid());
        }
    }

    // initialize namespace table.
    MessageContext.NamespaceUris = new NamespaceTable();
    MessageContext.NamespaceUris.Append(configuration.ApplicationUri);

    // assign an instance name.
    if (String.IsNullOrEmpty(configuration.ApplicationName) && InstanceCertificate != null)
    {
        configuration.ApplicationName = InstanceCertificate.GetNameInfo(X509NameType.DnsName, false);
    }

    // save the certificate validator.
    CertificateValidator = configuration.CertificateValidator;
}
```

#### 2.1.1. `LoadServerProperties`

该方法由具体的`Server`重写，基类只适合直接`new`一个`ServerProperty`对象返回，

这里就是由`EmptyServer`实现，都是一些非配置性的属性：

```C#
protected override ServerProperties LoadServerProperties()
{
    ServerProperties properties = new ServerProperties();

    properties.ManufacturerName = "OPC Foundation";
    properties.ProductName      = "Quickstart Empty Server";
    properties.ProductUri       = "http://opcfoundation.org/Quickstart/EmptyServer/v1.0";
    properties.SoftwareVersion  = Utils.GetAssemblySoftwareVersion();
    properties.BuildNumber      = Utils.GetAssemblyBuildNumber();
    properties.BuildDate        = Utils.GetAssemblyTimestamp();
    return properties; 
}   
```

#### 2.1.2  创建`ServiceMessageContext`对象

```C#
public ServiceMessageContext CreateMessageContext(bool clonedFactory=false)
{
    ServiceMessageContext messageContext = new ServiceMessageContext();
	//m_transportQuotas应该是一些通讯传输的限制参数
    if (m_transportQuotas != null)
    {
        messageContext.MaxArrayLength = m_transportQuotas.MaxArrayLength;
        messageContext.MaxByteStringLength = m_transportQuotas.MaxByteStringLength;
        messageContext.MaxStringLength = m_transportQuotas.MaxStringLength;
        messageContext.MaxMessageSize = m_transportQuotas.MaxMessageSize;
    }

    messageContext.NamespaceUris = new NamespaceTable();
    messageContext.ServerUris = new StringTable();
    if (clonedFactory)
    {
        messageContext.Factory = new EncodeableFactory(EncodeableFactory.GlobalFactory);
    }
    return messageContext;
}
```

### 2.2.  `InitializeRequestQueue`

```C#
protected void InitializeRequestQueue(ApplicationConfiguration configuration)
{
    //最小请求线程数、最大线程数，最大队列书
    int minRequestThreadCount = 10;
    int maxRequestThreadCount = 1000;
    int maxQueuedRequestCount = 2000;

    //从配置文件中获取参数配置
    if (configuration.ServerConfiguration != null)
    {
        minRequestThreadCount = configuration.ServerConfiguration.MinRequestThreadCount;
        maxRequestThreadCount = configuration.ServerConfiguration.MaxRequestThreadCount;
        maxQueuedRequestCount = configuration.ServerConfiguration.MaxQueuedRequestCount;
    }

    else if (configuration.DiscoveryServerConfiguration != null)
    {
        minRequestThreadCount = configuration.DiscoveryServerConfiguration.MinRequestThreadCount;
        maxRequestThreadCount = configuration.DiscoveryServerConfiguration.MaxRequestThreadCount;
        maxQueuedRequestCount = configuration.DiscoveryServerConfiguration.MaxQueuedRequestCount;
    }

    // 参数纠正
    if (maxRequestThreadCount < 100)
    {
        maxRequestThreadCount = 100;
    }

    if (maxQueuedRequestCount < 100)
    {
        maxQueuedRequestCount = 100;
    }

    if (m_requestQueue != null)
    {
        m_requestQueue.Dispose();
    }
	//初始化队列，RequestQueue是一个请求的队列容器
    m_requestQueue = new RequestQueue(this, minRequestThreadCount, maxRequestThreadCount, maxQueuedRequestCount);
}

```

`RequestQueue`是一个请求的队列容器,含有`Server`的引用以及一个代表自身状态的字段`m_stopped`

主要方法就是`ScheduleIncomingRequest`：开启一个线程，执行`server`的`ProcessRequest`方法

```C#

protected class RequestQueue : IDisposable
{

    public RequestQueue(ServerBase server, int minThreadCount, int maxThreadCount, int maxRequestCount)
    {
        m_server = server;
        m_stopped = false;
    }
    
    public void Dispose()
    {
        Dispose(true);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (disposing)
        {
            m_stopped = true;
        }
    }

    public void ScheduleIncomingRequest(IEndpointIncomingRequest request)
    {
        if (m_stopped)
        {
            request.OperationCompleted(null, StatusCodes.BadTooManyOperations);
        }
        else
        {
            Task.Run(() =>
                     {
                         m_server.ProcessRequest(request);
                     });
        }
    }
    private ServerBase m_server;
    private bool m_stopped;
}
```

### 2.3. `InitializeBaseAddresses`

```C#
private void InitializeBaseAddresses(ApplicationConfiguration configuration)
{
    BaseAddresses = new List<BaseAddress>();

    StringCollection sourceBaseAddresses = null;
    StringCollection sourceAlternateAddresses = null;
	//从配置文件中获取服务端地址等信息
    if (configuration.ServerConfiguration != null)
    {
        sourceBaseAddresses = configuration.ServerConfiguration.BaseAddresses;
        sourceAlternateAddresses = configuration.ServerConfiguration.AlternateBaseAddresses;
    }

    if (configuration.DiscoveryServerConfiguration != null)
    {
        sourceBaseAddresses = configuration.DiscoveryServerConfiguration.BaseAddresses;
        sourceAlternateAddresses = configuration.DiscoveryServerConfiguration.AlternateBaseAddresses;
    }

    if (sourceBaseAddresses == null)
    {
        return;
    }

    foreach (string baseAddress in sourceBaseAddresses)
    {
        //将字符串地址转换为url，存入BaseAddres对象
        BaseAddress address = new BaseAddress() { Url = new Uri(baseAddress) };

        //找到和baseAddress对应的alternateAddress协议匹配的记录,将alternateUrl缓存
        if (sourceAlternateAddresses != null)
        {
            foreach (string alternateAddress in sourceAlternateAddresses)
            {
                Uri alternateUrl = new Uri(alternateAddress);

                if (alternateUrl.Scheme == address.Url.Scheme)
                {
                    if (address.AlternateUrls == null)
                    {
                        address.AlternateUrls = new List<Uri>();
                    }

                    address.AlternateUrls.Add(alternateUrl);
                }
            }
        }
		
        //根据协议头，设置不同的ProfileUri
        switch (address.Url.Scheme)
        {
            case Utils.UriSchemeHttps:
                {
                    address.ProfileUri = Profiles.HttpsBinaryTransport;
                    address.DiscoveryUrl = address.Url;
                    break;
                }

            case Utils.UriSchemeOpcTcp:
                {
                    address.ProfileUri = Profiles.UaTcpTransport;
                    address.DiscoveryUrl = address.Url;
                    break;
                }
        }

        BaseAddresses.Add(address);
    }
}
```

### 2.4. `InitializeServiceHosts`

```C
protected override IList<Task> InitializeServiceHosts(
    ApplicationConfiguration          configuration, 
    out ApplicationDescription        serverDescription,
    out EndpointDescriptionCollection endpoints)
{
    serverDescription = null;
    endpoints = null;

    Dictionary<string, Task> hosts = new Dictionary<string, Task>();

    // 确保至少一个安全策略
    if (configuration.ServerConfiguration.SecurityPolicies.Count == 0)
    {                   
        configuration.ServerConfiguration.SecurityPolicies.Add(new ServerSecurityPolicy());
    }

    // 确保至少一个userToken策略
    if (configuration.ServerConfiguration.UserTokenPolicies.Count == 0)
    {                   
        UserTokenPolicy userTokenPolicy = new UserTokenPolicy();

        userTokenPolicy.TokenType = UserTokenType.Anonymous;
        userTokenPolicy.PolicyId  = userTokenPolicy.TokenType.ToString();

        configuration.ServerConfiguration.UserTokenPolicies.Add(userTokenPolicy);
    }

    //ApplicationDescription就是框架描述对象
    serverDescription = new ApplicationDescription();

    serverDescription.ApplicationUri = configuration.ApplicationUri;
    serverDescription.ApplicationName = new LocalizedText("en-US", configuration.ApplicationName);
    serverDescription.ApplicationType = configuration.ApplicationType;
    serverDescription.ProductUri = configuration.ProductUri;
    serverDescription.DiscoveryUrls = GetDiscoveryUrls();
	//通讯节点集合
    endpoints = new EndpointDescriptionCollection();
    IList<EndpointDescription> endpointsForHost = null;

    //创建uaTCP连接
    endpointsForHost = CreateUaTcpServiceHost(
        hosts,
        configuration,
        configuration.ServerConfiguration.BaseAddresses,
        serverDescription,
        configuration.ServerConfiguration.SecurityPolicies);

    endpoints.InsertRange(0, endpointsForHost);

    //如果有安全需要，创建HTTPs连接
    #if !NO_HTTPS
    endpointsForHost = CreateHttpsServiceHost(
        hosts,
        configuration,
        configuration.ServerConfiguration.BaseAddresses,
        serverDescription,
        configuration.ServerConfiguration.SecurityPolicies);

    endpoints.AddRange(endpointsForHost);
    #endif
    return new List<Task>(hosts.Values);
}
```

继续跟踪`ServerBase`的`CreateUaTcpServiceHost`

```c#
protected List<EndpointDescription> CreateUaTcpServiceHost(
    IDictionary<string, Task> hosts,
    ApplicationConfiguration configuration,
    IList<string> baseAddresses,
    ApplicationDescription serverDescription,
    List<ServerSecurityPolicy> securityPolicies)
{
    // generate a unique host name.
    string hostName = String.Empty;

    if (hosts.ContainsKey(hostName))
    {
        hostName = "/Tcp";
    }

    if (hosts.ContainsKey(hostName))
    {
        hostName += Utils.Format("/{0}", hosts.Count);
    }
    List<Uri> uris = new List<Uri>();
    EndpointDescriptionCollection endpoints = new EndpointDescriptionCollection();

    //创建EndpointConfiguration对象，主要是用来保存通讯相关参数，来自config文件的TransportQuotas节点
    EndpointConfiguration endpointConfiguration = EndpointConfiguration.Create(configuration);
    //本机名称
    string computerName = Utils.GetHostName();

    for (int ii = 0; ii < baseAddresses.Count; ii++)
    {
        //排除非opc.tcp开头的地址
        if (!baseAddresses[ii].StartsWith(Utils.UriSchemeOpcTcp, StringComparison.Ordinal))
        {
            continue;
        }
		//如果从字符串中解析的host是以localhost作为主机，则将本机计算机名称赋值给uri.host.
        UriBuilder uri = new UriBuilder(baseAddresses[ii]);
        if (String.Compare(uri.Host, "localhost", StringComparison.OrdinalIgnoreCase) == 0)
        {
            uri.Host = computerName;
        }

        uris.Add(uri.Uri);
		//。。。。
        //中间一大坨安全认证相关代码略过

        // create the UA-TCP stack listener.
        try
        {
            TransportListenerSettings settings = new TransportListenerSettings();

            settings.Descriptions = endpoints;
            settings.Configuration = endpointConfiguration;
            settings.CertificateValidator = configuration.CertificateValidator.GetChannelValidator();
            settings.NamespaceUris = this.MessageContext.NamespaceUris;
            settings.Factory = this.MessageContext.Factory;
            settings.ServerCertificate = this.InstanceCertificate;

            if (configuration.SecurityConfiguration.SendCertificateChain)
            {
                settings.ServerCertificateChain = this.InstanceCertificateChain;
            }

           	//生成一个监听器，这里注意下listener就是一个socket服务处理端，不是观察者模式中的监听者
            ITransportListener listener = new Opc.Ua.Bindings.UaTcpChannelListener();

            //open中开启socket监听，GetEndpointInstance是回调方法
            listener.Open(
                uri.Uri,
                settings,
                GetEndpointInstance(this));

            TransportListeners.Add(listener);
        }
        catch (Exception e)
        {
            Utils.Trace(e, "Could not load UA-TCP Stack Listener.");
            throw e;
        }
    }

    return endpoints;
}
```

#### 2.4.1 `TcpListener.start`

```C#
public void Start()
{
    lock (m_lock)
    {
        //端口检查
        int port = m_uri.Port;
        if (port <= 0 || port > UInt16.MaxValue)
        {
            port = Utils.UaTcpDefaultPort;
        }

        //通过socket创建tcp连接
        try
        {
            IPEndPoint endpoint = new IPEndPoint(IPAddress.Any, port);
            m_listeningSocket = new Socket(endpoint.AddressFamily, SocketType.Stream, ProtocolType.Tcp);
            SocketAsyncEventArgs args = new SocketAsyncEventArgs();
            args.Completed += OnAccept;
            //socket存储到userToken
            args.UserToken = m_listeningSocket;
            m_listeningSocket.Bind(endpoint);
            m_listeningSocket.Listen(Int32.MaxValue);
            //发起异步连接
            if (!m_listeningSocket.AcceptAsync(args))
            {
                OnAccept(null, args);
            }
        }
        catch (Exception ex)
        {
            m_listeningSocket = null;
            Utils.Trace("failed to create IPv4 listening socket: " + ex.Message);
        }

        // create IPv6 socket
        // 。。。略过
    }
}
```



#### 2.4.2 `OnAccept`

```C#
private void OnAccept(object sender, SocketAsyncEventArgs e)
{
    TcpServerChannel channel = null;
    bool repeatAccept = false;
    do
    {
        repeatAccept = false;
        lock (m_lock)
        {
            //检查ServerSocket
            Socket listeningSocket = e.UserToken as Socket;
            if (listeningSocket == null)
            {
                Utils.Trace("OnAccept: Listensocket was null.");
                e.Dispose();
                return;
            }

            // 检查socket运行状态
            if (e.AcceptSocket != null && e.SocketError == SocketError.Success)
            {
                try
                {
                    //TcpServerChannel对应每一个连接，用于管理连接通讯
                    channel = new TcpServerChannel(
                        m_listenerId,//这是serversocket的唯一标识
                        this,//Listener对象
                        m_bufferManager,
                        m_quotas,
                        m_serverCertificate,
                        m_serverCertificateChain,
                        m_descriptions);

                    //该回调方法就是之前设置的GetEndpointInstance方法
                    if (m_callback != null)
                    {
                        channel.SetRequestReceivedCallback(new TcpChannelRequestEventHandler(OnRequestReceived));
                    }

                    // m_lastChannelId：每个连接的唯一标识
                    // e.AcceptSocket:客户端连接对象
                    channel.Attach(++m_lastChannelId, e.AcceptSocket);

                    // save the channel for shutdown and reconnects.
                    m_channels.Add(m_lastChannelId, channel);

                }
                catch (Exception ex)
                {
                    Utils.Trace(ex, "Unexpected error accepting a new connection.");
                }
            }

            e.Dispose();

            //等待处理新的连接
            if (e.SocketError != SocketError.OperationAborted)
            {
                // go back and wait for the next connection.
                try
                {
                    e = new SocketAsyncEventArgs();
                    e.Completed += OnAccept;
                    e.UserToken = listeningSocket;
                    if (!listeningSocket.AcceptAsync(e))
                    {
                        repeatAccept = true;
                    }
                }
                catch (Exception ex)
                {
                    Utils.Trace(ex, "Unexpected error listening for a new connection.");
                }
            }
        }
    } while (repeatAccept);
}
```

#### 2.4.3. `TcpServerChannel.Attach`

继续观察下`channel.Attach(++m_lastChannelId, e.AcceptSocket);`方法

```C#
public void Attach(uint channelId, Socket socket)
{
    if (socket == null)
    {
        throw new ArgumentNullException(nameof(socket));
    }

    lock (DataLock)
    {
        //IMessageSocket对象主要处理连接的读写
        if (Socket != null)
        {
            throw new InvalidOperationException("Channel is already attached to a socket.");
        }

        ChannelId = channelId;
        State = TcpChannelState.Connecting;

        //关注TcpMessageSocket，会注册读写相关回调
        Socket = new TcpMessageSocket(this, socket, BufferManager, Quotas.MaxBufferSize);
        Utils.Trace("TCPSERVERCHANNEL SOCKET ATTACHED: {0:X8}, ChannelId={1}", Socket.Handle, ChannelId);
        Socket.ReadNextMessage();

        // automatically clean up the channel if no hello received.
        StartCleanupTimer(StatusCodes.BadTimeout);
    }
}
```

#### 2..4.4.  `TcpMessageSocket`

```C#
public class TcpMessageSocket : IMessageSocket
{
        /// <summary>
        /// Creates an unconnected socket.
        /// </summary>
        public TcpMessageSocket(
        IMessageSink sink,
        BufferManager bufferManager,
        int receiveBufferSize)
    {
        if (bufferManager == null)
        {
            throw new ArgumentNullException(nameof(bufferManager));
        }

        m_sink = sink;
        m_socket = null;
        m_bufferManager = bufferManager;
        m_receiveBufferSize = receiveBufferSize;
        m_incomingMessageSize = -1;
        //数据读取的回调函数
        m_ReadComplete = new EventHandler<SocketAsyncEventArgs>(OnReadComplete);
    }
    
    private void OnReadComplete(object sender, SocketAsyncEventArgs e)
    {
        lock (m_readLock)
        {
            ServiceResult error = null;
			//此处省略一些无关代码
           error = DoReadComplete(e);

            if (ServiceResult.IsBad(error))
            {
                if (m_receiveBuffer != null)
                {
                    m_bufferManager.ReturnBuffer(m_receiveBuffer, "OnReadComplete");
                    m_receiveBuffer = null;
                }

                if (m_sink != null)
                {
                    m_sink.OnReceiveError(this, error);
                }
            }
        }
    }
    
    public void ReadNextMessage()
    {
        lock (m_readLock)
        {
            if (m_receiveBuffer == null)
            {
                //此处通过BufferManger读取字节数组，在结尾加上0x5a作为分隔
                m_receiveBuffer = m_bufferManager.TakeBuffer(m_receiveBufferSize, "ReadNextMessage");
            }

            // read the first 8 bytes of the message which contains the message size.          		  //读取前8个字节：表示消息类型
            m_bytesReceived = 0;
            m_bytesToReceive = TcpMessageLimits.MessageTypeAndSize;
            m_incomingMessageSize = -1;

            ReadNextBlock();
        }
    }
    
    private void ReadNextBlock()
    {
        Socket socket = null;
		//....
        BufferManager.LockBuffer(m_receiveBuffer);

        ServiceResult error = ServiceResult.Good;
        SocketAsyncEventArgs args = new SocketAsyncEventArgs();
        try
        {
            args.SetBuffer(m_receiveBuffer, m_bytesReceived, m_bytesToReceive - m_bytesReceived);
            args.Completed += m_ReadComplete;
            //发起异步读取数据
            if (!socket.ReceiveAsync(args))
            {
                // I/O completed synchronously
                if (args.SocketError != SocketError.Success)
                {
                    throw ServiceResultException.Create(StatusCodes.BadTcpInternalError, args.SocketError.ToString());
                }
                else
                {
                    //同步操作则开线程执行回调
                    Task.Run(() => m_ReadComplete(null, args));
                }
            }
        }
        catch (ServiceResultException sre)
        {
            args.Dispose();
            BufferManager.UnlockBuffer(m_receiveBuffer);
            throw sre;
        }
        catch (Exception ex)
        {
            args.Dispose();
            BufferManager.UnlockBuffer(m_receiveBuffer);
            throw ServiceResultException.Create(StatusCodes.BadTcpInternalError, ex, "BeginReceive failed.");
        }
    }
}

```

从上面的流程可以得知，`ReadNextMessage`会调用`ReadNextBlock`，

而`ReadNextBlock`才是真正执行异步读取的地方

我们再回到数据读取后的处理方法`DoReadComplete`

#### 2.4.5. `DoReadComplete`

```c#
/// <summary>
/// Handles a read complete event.
/// </summary>
private ServiceResult DoReadComplete(SocketAsyncEventArgs e)
{
    //可读字节数
    int bytesRead = e.BytesTransferred;

    lock (m_socketLock)
    {
        BufferManager.UnlockBuffer(m_receiveBuffer);
    }

    Utils.TraceDebug("Bytes read: {0}", bytesRead);

    //可读数为0，远端关闭
    if (bytesRead == 0)
    {
        // Remote end has closed the connection

        // free the empty receive buffer.
        if (m_receiveBuffer != null)
        {
            m_bufferManager.ReturnBuffer(m_receiveBuffer, "DoReadComplete");
            m_receiveBuffer = null;
        }

        return ServiceResult.Create(StatusCodes.BadConnectionClosed, "Remote side closed connection");
    }

    m_bytesReceived += bytesRead;

    // 字节数不足8，无法解析成消息类型，继续读取
    if (m_bytesReceived < m_bytesToReceive)
    {
        ReadNextBlock();

        return ServiceResult.Good;
    }

    // m_incomingMessageSize初始值=-1
    if (m_incomingMessageSize < 0)
    {
        //从第4个字节往后4个字节表示长度
        m_incomingMessageSize = BitConverter.ToInt32(m_receiveBuffer, 4);

        if (m_incomingMessageSize <= 0 || m_incomingMessageSize > m_receiveBufferSize)
        {
            Utils.Trace(
                "BadTcpMessageTooLarge: BufferSize={0}; MessageSize={1}",
                m_receiveBufferSize,
                m_incomingMessageSize);

            return ServiceResult.Create(
                StatusCodes.BadTcpMessageTooLarge,
                "Messages size {1} bytes is too large for buffer of size {0}.",
                m_receiveBufferSize,
                m_incomingMessageSize);
        }

        // set up buffer for reading the message body.
        m_bytesToReceive = m_incomingMessageSize;
		//继续读取数据body
        ReadNextBlock();

        return ServiceResult.Good;
    }

    if (m_sink != null)
    {
        try
        {
            // send notification (implementor responsible for freeing buffer) on success.
            ArraySegment<byte> messageChunk = new ArraySegment<byte>(m_receiveBuffer, 0, m_incomingMessageSize);

            // must allocate a new buffer for the next message.
            m_receiveBuffer = null;
			//返回数据处理
            m_sink.OnMessageReceived(this, messageChunk);
        }
        catch (Exception ex)
        {
            Utils.Trace(ex, "Unexpected error invoking OnMessageReceived callback.");
        }
    }

    // free the receive buffer.
    if (m_receiveBuffer != null)
    {
        m_bufferManager.ReturnBuffer(m_receiveBuffer, "DoReadComplete");
        m_receiveBuffer = null;
    }

    // start receiving next message.
    ReadNextMessage();

    return ServiceResult.Good;
}
```

#### 2.4.6. `UaSCUaBinaryChannel`

上述 `m_sink.OnMessageReceived(this, messageChunk);`

实际是由`TCPChannel`的基类`UaSCUaBinaryChannel`来处理的，且实现了`IMessageSink`接口

```C#
public virtual void OnMessageReceived(IMessageSocket source, ArraySegment<byte> message)
{
    lock (DataLock)
    {
        try
        {
            //读取4个字节的消息类型
            uint messageType = BitConverter.ToUInt32(message.Array, message.Offset);

            Utils.TraceDebug("{1} Message Received: {0} bytes", message.Count, messageType);
			//根据不同的消息类型处理
            if (!HandleIncomingMessage(messageType, message))
            {
                BufferManager.ReturnBuffer(message.Array, "OnMessageReceived");
            }
        }
        catch (Exception e)
        {
            HandleMessageProcessingError(e, StatusCodes.BadTcpInternalError, "An error occurred receiving a message.");
            BufferManager.ReturnBuffer(message.Array, "OnMessageReceived");
        }
    }
}

```

看`TcpServerChannel`的实现

```C#
protected override bool HandleIncomingMessage(uint messageType, ArraySegment<byte> messageChunk)
{
    lock (DataLock)
    {
        m_responseRequired = true;

        try
        {
            // process a response.
            if (TcpMessageType.IsType(messageType, TcpMessageType.Message))
            {
                //Utils.Trace("Channel {0}: ProcessRequestMessage", ChannelId);
                return ProcessRequestMessage(messageType, messageChunk);
            }

            // check for hello.
            if (messageType == TcpMessageType.Hello)
            {
                //Utils.Trace("Channel {0}: ProcessHelloMessage", ChannelId);
                return ProcessHelloMessage(messageChunk);
            }

            // process open secure channel repsonse.
            if (TcpMessageType.IsType(messageType, TcpMessageType.Open))
            {
                //Utils.Trace("Channel {0}: ProcessOpenSecureChannelRequest", ChannelId);
                return ProcessOpenSecureChannelRequest(messageType, messageChunk);
            }

            // process close secure channel response.
            if (TcpMessageType.IsType(messageType, TcpMessageType.Close))
            {
                //Utils.Trace("Channel {0}: ProcessCloseSecureChannelRequest", ChannelId);
                return ProcessCloseSecureChannelRequest(messageType, messageChunk);
            }

            // invalid message type - must close socket and reconnect.
            ForceChannelFault(
                StatusCodes.BadTcpMessageTypeInvalid,
                "The server does not recognize the message type: {0:X8}.",
                messageType);

            return false;
        }
        finally
        {
            m_responseRequired = false;
        }
    }
}
```







### 2.5 `StartApplication`

具体基类没有实现，主要看`StandServer`

```C#
protected override void StartApplication(ApplicationConfiguration configuration)
{
    base.StartApplication(configuration);

    lock (m_lock)
    {
        try
        {
            // create the datastore for the instance.
            m_serverInternal = new ServerInternalData(
                ServerProperties, 
                configuration, 
                MessageContext,
                new CertificateValidator(),
                InstanceCertificate);

            // create the manager responsible for providing localized string resources.                    
            ResourceManager resourceManager = CreateResourceManager(m_serverInternal, configuration);

            // create the manager responsible for incoming requests.
            RequestManager requestManager = CreateRequestManager(m_serverInternal, configuration);

            // create the master node manager.
            MasterNodeManager masterNodeManager = CreateMasterNodeManager(m_serverInternal, configuration);

            // add the node manager to the datastore. 
            m_serverInternal.SetNodeManager(masterNodeManager);

            // put the node manager into a state that allows it to be used by other objects.
            masterNodeManager.Startup();

            // create the manager responsible for handling events.
            EventManager eventManager = CreateEventManager(m_serverInternal, configuration);

            // creates the server object. 
            m_serverInternal.CreateServerObject(
                eventManager,
                resourceManager, 
                requestManager);

            // do any additional processing now that the node manager is up and running.
            OnNodeManagerStarted(m_serverInternal);

            // create the manager responsible for aggregates.
            m_serverInternal.AggregateManager = CreateAggregateManager(m_serverInternal, configuration);

            // start the session manager.
            SessionManager sessionManager = CreateSessionManager(m_serverInternal, configuration);
            sessionManager.Startup();

            // start the subscription manager.
            SubscriptionManager subscriptionManager = CreateSubscriptionManager(m_serverInternal, configuration);
            subscriptionManager.Startup();

            // add the session manager to the datastore. 
            m_serverInternal.SetSessionManager(sessionManager, subscriptionManager);

            ServerError = null;

            // setup registration information.
            lock (m_registrationLock)
            {
                m_maxRegistrationInterval = configuration.ServerConfiguration.MaxRegistrationInterval;

                ApplicationDescription serverDescription = ServerDescription;

                m_registrationInfo = new RegisteredServer();

                m_registrationInfo.ServerUri = serverDescription.ApplicationUri;
                m_registrationInfo.ServerNames.Add(serverDescription.ApplicationName);
                m_registrationInfo.ProductUri = serverDescription.ProductUri;
                m_registrationInfo.ServerType = serverDescription.ApplicationType;
                m_registrationInfo.GatewayServerUri = null;
                m_registrationInfo.IsOnline = true;
                m_registrationInfo.SemaphoreFilePath = null;

                // add all discovery urls.
                string computerName = Utils.GetHostName();

                for (int ii = 0; ii < BaseAddresses.Count; ii++)
                {
                    UriBuilder uri = new UriBuilder(BaseAddresses[ii].DiscoveryUrl);

                    if (String.Compare(uri.Host, "localhost", StringComparison.OrdinalIgnoreCase) == 0)
                    {
                        uri.Host = computerName;
                    }

                    m_registrationInfo.DiscoveryUrls.Add(uri.ToString());
                }

                // build list of registration endpoints.
                m_registrationEndpoints = new ConfiguredEndpointCollection(configuration);

                EndpointDescription endpoint = configuration.ServerConfiguration.RegistrationEndpoint;

                if (endpoint == null)
                {
                    endpoint = new EndpointDescription();
                    endpoint.EndpointUrl = Utils.Format(Utils.DiscoveryUrls[0], "localhost");
                    endpoint.SecurityLevel = ServerSecurityPolicy.CalculateSecurityLevel(MessageSecurityMode.SignAndEncrypt, SecurityPolicies.Basic256Sha256);
                    endpoint.SecurityMode = MessageSecurityMode.SignAndEncrypt;
                    endpoint.SecurityPolicyUri = SecurityPolicies.Basic256Sha256;
                    endpoint.Server.ApplicationType = ApplicationType.DiscoveryServer;
                }

                m_registrationEndpoints.Add(endpoint);

                m_minRegistrationInterval  = 1000;
                m_lastRegistrationInterval = m_minRegistrationInterval;

                // start registration timer.
                if (m_registrationTimer != null)
                {
                    m_registrationTimer.Dispose();
                    m_registrationTimer = null;
                }

                if (m_maxRegistrationInterval > 0)
                {
                    m_registrationTimer = new Timer(OnRegisterServer, this, m_minRegistrationInterval, Timeout.Infinite);
                }
            }

            // set the server status as running.
            SetServerState(ServerState.Running);

            // all initialization is complete.
            OnServerStarted(m_serverInternal); 

            // monitor the configuration file.
            if (!String.IsNullOrEmpty(configuration.SourceFilePath))
            {
                m_configurationWatcher = new ConfigurationWatcher(configuration);
                m_configurationWatcher.Changed += new EventHandler<ConfigurationWatcherEventArgs>(this.OnConfigurationChanged);
            }

            CertificateValidator.CertificateUpdate += OnCertificateUpdate;
        }
        catch (Exception e)
        {
            Utils.Trace(e, "Unexpected error starting application");
            m_serverInternal = null;
            ServiceResult error = ServiceResult.Create(e, StatusCodes.BadInternalError, "Unexpected error starting application");
            ServerError = error;
            throw new ServiceResultException(error);
        }
    }
}
```



