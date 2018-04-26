## 网络通信基础 ##

### OSI七层模型  ###

OSI(Open System Interconnection 开放系统互联)

![](http://upload-images.jianshu.io/upload_images/1604627-21872f2e1350ad1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总得来说就是将你需要发送的数据通过一层层的封装最后一比特流的形式发送到目的进程。

### TCP/IP协议 四层模型 ###

![](http://upload-images.jianshu.io/upload_images/1604627-799ac7896ddc3da3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**这里的TCP/IP指的可不是tcp/ip两种协议，而是一种网络模型**

![](http://upload-images.jianshu.io/upload_images/1604627-aaa799c7b79fef68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

各层级对应的常用的协议，可以看出Http协议和ftp协议都是属于应用层的。而我们常用的socket是在应用层和传输层之间的。

### TCP/IP模型 ###


> 网络接口层

1. 封装数据/解封数据
2. 控制帧传输
3. 流量控制

> 网络层

我们熟知的主要是ip协议，网络层之上的协议包括TCP.UDP等协议数据都是以IP数据报格式传输的。

IP协议主要负责网络主机的定位，数据传输的路由，由IP地址可以唯一地确定Internet上的一台主机

**ip协议只管将数据从源IP传输到目标IP，其他的诸如数据安全都不关注。**

> 传输层

主要协议有 **TCP/UDP协议**

不同的应用进程使用不同的协议。

![](http://upload-images.jianshu.io/upload_images/1604627-aaa6754e73943193.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**UDP协议**

用户数据报协议  
传输速度快，开销小，无连接，无阻塞，不可靠，不稳定。  
支持一对一，一对多，多对一，多对多通信

**TCP协议**

传输控制协议  

点对点连接，可靠有序，全双工通信(双方可以互发数据)，面向字节流  

**TCP协议建立连接的三次握手**

1. 客户端发送连接请求报文段
2. 服务端收到连接请求，并返回确认
3. 客户端再次发送确认报文段(指自己已经收到服务端的确认报文)，**这个时候发送的报文段已经可以携带数据**

**TCP协议断开连接的四次挥手**

1. 客户端发送断开请求
2. 服务器收到断开请求，并返回确认
3. 服务器发送断开连接的请求
4. 客户端返回确认

> socket

位于传输层之上，应用层之下，是对传输层的封装。

![](http://upload-images.jianshu.io/upload_images/1604627-4c7d183759ce8284.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

两个进程如何通信最主要的前提是能够唯一标示一个进程，在本地进程通讯中我们可以使用pid来唯一标示一个进程。

但是在网络中两个进程的pid有很大几率会冲突。所以通过ip地址+协议+端口号唯一标示网络中的一个进程。

IP层的ip地址可以唯一标示主机，而TCP层协议和端口号可以唯一标示主机的一个进程

socket是TCP/IP的抽象和封装，简化TCP/IP的操作难度，socket主要用来进行进程间的通信。

**实际上网络间的通信还是基于TCP/IP的协议。**

> 应用层

![](http://upload-images.jianshu.io/upload_images/1604627-aaa6754e73943193.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不同的应用对应具体的需求，同时也需要使用不同的应用层协议。之后会对http协议进行进一步解读。

### socket解读 ###

> 概念解释

Socket非常类似于电话插座。以一个国家级电话网为例，电话的通话双方相当于相互通信的2个进程，区号是它的网络地址；区内一个单位的交换机相当于一台主机，主机分配给每个用户的局内号码相当于Socket号。任何用户在通话之前，首先要占有一部电话机，相当于申请一个Socket；同时要知道对方的号码，相当于对方有一个固定的Socket。然后向对方拨号呼叫，相当于发出连接请求（假如对方不在同一区内，还要拨对方区号，相当于给出网络地址）。假如对方在场并空闲（相当于通信的另一主机开机且可以接受连接请求），拿起电话话筒，双方就可以正式通话，相当于连接成功。双方通话的过程，是一方向电话机发出信号和对方从电话机接收信号的过程，相当于向Socket发送数据和从socket接收数据。通话结束后，一方挂起电话机相当于关闭Socket，撤消连接。

socket又称套接字，底层建立连接通道，通过套接字建立连接。

socket是支持TCP/IP协议的网络通信的基本操作单元。它是网络通信过程中断点的抽象标示，包含进行网络通信的必须得五种信息

1. 连接协议(tcp/ip)
2. 源ip
3. 源端口
4. 目标ip
5. 目标端口

**socket的通信流程图**

![](http://upload-images.jianshu.io/upload_images/1803860-c105e0457c718495.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### HTTP协议解读 ###

http协议属于应用层协议，也是离数据最近的一层。

http发送是以报文的形式发送，报文的每一个地段都是ASCII码串。

![](http://upload-images.jianshu.io/upload_images/1604627-94e898d708e2086e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Http请求报文，由4部分组成，请求行，请求头部，空行，请求体。

> 请求行

- 请求方法:GET,POST,HEAD(**只返回响应头**),PUT等
- URL:www.baidu.com
- 版本协议:HTTP/1.1  

> 请求头

键值对的形式

- User-Agent：产生请求的浏览器类型。
- Accept：客户端可识别的内容类型列表。
- Host：请求的主机名，允许多个域名同处一个IP地址，即虚拟主机。
- Accept-Language：客户端可接受的自然语言。
- Accept-Encoding：客户端可接受的编码压缩格式。
- Accept-Charset：可接受的应答的字符集。
- connection：连接方式(close 或 keepalive)。
- Cookie：存储于客户端扩展字段，向同一域名的服务端发送属于该域的cookie。

> 空行

分割请求头和请求体的作用，表示接下来的内容为请求体。

> 请求体

主要是http请求的参数，GET请求中不需要这部分内容。

**实例：username=11111&password=00000**
这部分内容在GET请求中是加在URL后的。

### 响应报文 ###

响应报文跟请求报文差不多，状态行，响应头，空行，响应体。

![](http://upload-images.jianshu.io/upload_images/1604627-78b7339555c4840a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 状态行

**状态码**：三位数字  

- 1xx：表示服务器已接收了客户端请求，客户端可继续发送请求;  
- 2xx：表示服务器已成功接收到请求并进行处理;  
- 3xx：表示服务器要求客户端重定向;  
- 4xx：表示客户端的请求有非法内容;  
- 5xx：表示服务器未能正常处理客户端的请求而出现意外错误;

**常见状态码:**

- 200 OK：客户端请求成功。  
- 400 Bad Request：客户端请求有语法错误，不能被服务器所理解。  
- 401 Unauthorized：请求未经授权，这个状态代码必须和WWW-Authenticate报头域一起使用。  
- 403 Forbidden：服务器收到请求，但是拒绝提供服务。  
- 404 Not Found：请求资源不存在，举个例子：输入了错误的URL。  
- 500 Internal Server Error：服务器发生不可预期的错误。  
- 503 Server Unavailable：服务器当前不能处理客户端的请求，一段时间后可能恢复正常，举个例子：HTTP/1.1 200 OK（CRLF）。  

> 响应头

- Location：Location响应报头域用于重定向接受者到一个新的位置。例如：客户端所请求的页面已不存在原先的位置，为了让客户端重定向到这个页面新的位置，服务器端可以发回Location响应报头后使用重定向语句，让客户端去访问新的域名所对应的服务器上的资源;
- Server：Server 响应报头域包含了服务器用来处理请求的软件信息及其版本。它和 User-Agent 请求报头域是相对应的，前者发送服务器端软件的信息，后者发送客户端软件(浏览器)和操作系统的信息。
- Vary：指示不可缓存的请求头列表;
- Connection：连接方式;对于请求来说：close(告诉 WEB 服务器或者代理服务器，在完成本次请求的响应后，断开连接，不等待本次连接的后续请求了)。keepalive(告诉WEB服务器或者代理服务器，在完成本次请求的响应后，保持连接，等待本次连接的后续请求);
- WWW-Authenticate：WWW-Authenticate响应报头域必须被包含在401 (未授权的)响应消息中，这个报头域和前面讲到的Authorization 请求报头域是相关的，当客户端收到 401 响应消息，就要决定是否请求服务器对其进行验证。如果要求服务器对其进行验证，就可以发送一个包含了Authorization 报头域的请求;

> 响应体

服务器返回的文本信息。  

### HTTP与HTTPS的区别 ###
> HTTPS是一种通过计算机网络进行安全通信的传输协议。HTTPS经由HTTP进行通信，但利用SSL/TLS来加密数据包。HTTPS开发的主要目的，是提供对网站服务器的身份 认证，保护交换数据的隐私与完整性。

HTTPS与HTTP相比,文字上只多了一个S,实际上在应用层之下多了一层**安全层**

![](https://upload-images.jianshu.io/upload_images/1604627-2ae010f5d1f167b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图所示,HTTPS比HTPP就多了一个安全层，而这个安全层具体都做了些什么呢？

1. 交换协议版本号
1. 选择一个两端都了解的密码
1. 对两端的身份进行认证
1. 生成临时的会话密钥，以便加密信道。

### WebSocket ###

> WebSocket是一种在单个TCP连接上进行全双工通讯的协议，它使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在WebSocket API中，浏览器和服务器只需要完成一次握
> 手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

为什么需要WebSocket，因为 HTTP 协议有一个缺陷：通信只能由客户端发起。而WebSocket可以实现双向通信。一般来说WebSocket是用来实现双工通信的长连接的。HTTP想要达到
这种效果，一般会通过轮询或者long poll来实现，这样比较占用资源且非常被动。

![16024c0e546fbe0a.webp.jpg](https://upload-images.jianshu.io/upload_images/1604627-9a5a368907c89df4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ps:Okhttp已支持WebSocket




**参看链接**

[漫谈网络通信](http://www.cnblogs.com/ivehd/p/networking.html)

[请求响应头各参数处理](http://www.cnblogs.com/ludashi/p/6237340.html)

[Http请求码的完整列表](http://blog.csdn.net/u011240877/article/details/46604973)

[从输入URL到页面加载完的过程中发生的事](http://www.guokr.com/question/554991/)

[Android网络编程：基础理论汇总](https://juejin.im/post/5a2614b8f265da432652af7d)



### 计划 ###

浅谈Android网络请求的前世今生-元基础HttpConnection  

浅谈Android网络请求的前世今生-先驱者xUtils

浅谈Android网络请求的前世今生-官方示范Volley 

浅谈Android网络请求的前世今生-变革者okhttp 




