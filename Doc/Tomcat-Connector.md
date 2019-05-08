
Connector内包含一个重要属性：<br>
1. ProtocolHandler

Connector自身的init，start均是调用ProtocolHandler的init，start


ProtocolHandler的实现方法：
AbstractProtocol
    |----AbstractAjpProtocol

    |____AbstractHttp11Protocol


AbstractProtocol包含两个重要属性<br>
1.AbstractEndpoint
2.Adapter


Endpoint中包含Acceptor、Poller
Poller处理完调用SocketProcessor的doRun方法
doRun方法中调用GetHandler.Process，GetHandler是外部类AbstractEndPoint的方法
ConnectionHandler作为实现类
调用Http11Process的process方法

Process调用ProtocolHandler.connectionhandler的process方法