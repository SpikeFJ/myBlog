虽然每个Channel都有一个关联的Socket对象；但并非所有的Socket都有一个关联的通道。传统方式创建了一个Socket对象，就不会有关联的SocketChannel并且它的getChannel总是返回null。


# ServerSocketChannel

由于ServerSocketChannel没有bind( )方法，因此有必要取出对等的socket并使用它来绑定到一
个端口以开始监听连接
```java
ServerSocketChannel ssc = ServerSocketChannel.open( );
ServerSocket serverSocket = ssc.socket( );
serverSocket.bind (new InetSocketAddress (1234));
```

ServerSocketChannel也有accept( )方法
1. 如果在ServerSocket上调用accept( )方法，那么总是阻塞并返回一个java.net.Socket对象
2. 如果在ServerSocketChannel上调用accept( )
方法则会返回SocketChannel类型的对象，返回的对象能够在非阻塞模式下运行

**当没有传入连接在等待时，ServerSocketChannel.accept( )会立即返
回null。正是这种检查连接而不阻塞的能力实现了可伸缩性并降低了复杂性**

## SocketChannel
```java
SocketChannel socketChannel =
SocketChannel.open (new InetSocketAddress ("somehost", somePort));
```
等价于
```java
SocketChannel socketChannel = SocketChannel.open( );
socketChannel.connect (new InetSocketAddress ("somehost", somePort));
```

SocketChannel的`connect`没法指定超时。`connect`方法在阻塞模式下，发起对请求地址的链接并立即返回。

如果返回值是true，说明连接立即建立了；
如果连接不能立即建立，`connect`会返回`false`且并发地继续连接建立过程。

可以调用`finishConnect`来完成连接过程，在非阻塞模式下有可能有以下几种情况：
1. `connect`尚未调用，则抛出`NoConnectionPendingException`异常
2. 连接正在建立，尚未完成。什么都不会发生，且方法立即返回`false`
3. 调用`connect`之后又切换回阻塞模式，则会阻塞直到连接建立完成为止。接着方法就会返回`true`
4. 初次调用`connect`或最后一次调用`finishConnect`之后，连接建立过程已经完成。那么`SocketChannel`对象的内部状态将被更新到已连接状态，`finishConnect`方法会返回`true`，然后`SocketChannel`对象就可以用来传输数据了。**??**
5. 连接已经建立。那么什么都不会发生，`finishConnect`方法返回`true`

下面是创建连接示例示例：
```java
InetSocketAddress addr = new InetSocketAddress (host, port);
SocketChannel sc = SocketChannel.open( );
sc.configureBlocking (false);
sc.connect (addr);
while ( ! sc.finishConnect( )) {
doSomethingElse( );
}
doSomethingWithChannel (sc);
sc.close( );
```
# DatagramChannel
正如SocketChannel模拟连接导向的流协议（如TCP/IP），DatagramChannel则模拟包导向的
无连接协议（如UDP/IP）

DatagramChannel对象既可以充当服务器（监听者）也可以充当客户端（发送者）