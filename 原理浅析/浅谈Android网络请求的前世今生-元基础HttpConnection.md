之前写过一篇关于网络请求相关的文章,主要关于一些网络基础.这篇则重点讲一讲Android下httpUrlConnect的内容。

### 相关链接 ###
[浅谈Android网络通信的前世今生--网络基础](https://www.jianshu.com/p/13b96d3f29c3)


### Android下的Http client ###

Android提供了三种Http client:  

1. HttpURLConnection  
2. Apache HttpClient  
3. okHttp

> HttpURLConnection

HttpUrlConnection是JDK里提供的联网API，我们知道Android SDK是基于Java的，所以当然优先考虑HttpUrlConnection这种最原始最基本的API，其实大多数开源的联网框架基本上也是基于JDK的HttpUrlConnection进行的封装罢了

> HttpClient(不建议使用)

HttpClient是开源组织Apache提供的Java请求网络框架，其最早是为了方便Java服务器开发而诞生的，是对JDK中的HttpUrlConnection各API进行了封装和简化，提高了性能并且降低了调用API的繁琐，不过官方已经不建议使用。

> okhttp

**OKHttp是现在主流应用使用的网络请求方式, 用来交换数据和内容, 有效的使用OKHttp可以使你的APP变的更快和减少流量的使用。**

- 支持SPDY,可以合并多个到同一个主机的请求
- 使用连接池技术减少请求的延迟(如果SPDY是可用的话)
- 使用GZIP压缩减少传输的数据量
- 缓存响应避免重复的网络请求

当你的网络出现拥挤的时候,就是OKHttp大显身手的时候,它可以避免常见的网络问题,如果你的服务是部署在不同的IP上面的,如果第一个连接失败,OkHTtp会尝试其他的连接。这对现在IPv4+IPv6中常见的把服务冗余部署在不同的数据中心上也是很有必要的。**OkHttp将使用现在TLS特性(SNI ALPN)来初始化新的连接，如果握手失败,将切换到TLS 1.0**。

目前使用率最高的当属okHttp,不过我们这篇文章还是要先讲讲Android下网络请求的先辈 `HttpUrlConnect`,**本文基于Android API26**

### 简单使用 ###
	
	URL url=new URL("www.baidu.com");
    HttpURLConnection conn= (HttpURLConnection) url.openConnection();
    InputStream in = new BufferedInputStream(conn.getInputStream());

**其中关键的两个过程**    
1.openConnection() 建立tcp连接  
2.getInputSteam() 发送http请求,并获取返回流

> openConnection()

**URL.java**

	public URLConnection openConnection() throws java.io.IOException {
        return handler.openConnection(this);
    }

其中 handler是URLStreamHandler的实例,handler的创建是在URL的构造函数中,其中调用了`getURLStreamHandler()`方法、

	static URLStreamHandler getURLStreamHandler(String protocol) {

        URLStreamHandler handler = handlers.get(protocol);
        if (handler == null) {

            boolean checkedWithFactory = false;

            // Use the factory (if any)
            if (factory != null) {
                handler = factory.createURLStreamHandler(protocol);
                checkedWithFactory = true;
            }

            // Try java protocol handler
            if (handler == null) {
                final String packagePrefixList = System.getProperty(protocolPathProp,"");
                StringTokenizer packagePrefixIter = new StringTokenizer(packagePrefixList, "|");

                while (handler == null &&
                       packagePrefixIter.hasMoreTokens()) {

                    String packagePrefix = packagePrefixIter.nextToken().trim();
                    try {
                        String clsName = packagePrefix + "." + protocol +
                          ".Handler";
                        Class<?> cls = null;
                        try {
                            ClassLoader cl = ClassLoader.getSystemClassLoader();
                            cls = Class.forName(clsName, true, cl);
                        } catch (ClassNotFoundException e) {
                            ClassLoader contextLoader = Thread.currentThread().getContextClassLoader();
                            if (contextLoader != null) {
                                cls = Class.forName(clsName, true, contextLoader);
                            }
                        }
                        if (cls != null) {
                            handler  =
                              (URLStreamHandler)cls.newInstance();
                        }
                    } catch (ReflectiveOperationException ignored) {
                    }
                }
            }

            // Fallback to built-in stream handler.
            // Makes okhttp the default http/https handler
            if (handler == null) {
                try {
                    // BEGIN Android-changed
                    // Use of okhttp for http and https
                    // Removed unnecessary use of reflection for sun classes
                    if (protocol.equals("file")) {
                        handler = new sun.net.www.protocol.file.Handler();
                    } else if (protocol.equals("ftp")) {
                        handler = new sun.net.www.protocol.ftp.Handler();
                    } else if (protocol.equals("jar")) {
                        handler = new sun.net.www.protocol.jar.Handler();
                    } else if (protocol.equals("http")) {
                        handler = (URLStreamHandler)Class.
                            forName("com.android.okhttp.HttpHandler").newInstance();
                    } else if (protocol.equals("https")) {
                        handler = (URLStreamHandler)Class.
                            forName("com.android.okhttp.HttpsHandler").newInstance();
                    }
                    // END Android-changed
                } catch (Exception e) {
                    throw new AssertionError(e);
                }
            }

            synchronized (streamHandlerLock) {

                URLStreamHandler handler2 = null;

                // Check again with hashtable just in case another
                // thread created a handler since we last checked
                handler2 = handlers.get(protocol);

                if (handler2 != null) {
                    return handler2;
                }

                // Check with factory if another thread set a
                // factory since our last check
                if (!checkedWithFactory && factory != null) {
                    handler2 = factory.createURLStreamHandler(protocol);
                }

                if (handler2 != null) {
                    // The handler from the factory must be given more
                    // importance. Discard the default handler that
                    // this thread created.
                    handler = handler2;
                }

                // Insert this handler into the hashtable
                if (handler != null) {
                    handlers.put(protocol, handler);
                }

            }
        }

        return handler;

    }

1. 从handlers中取
2. 如果URLStreamHandlerFactory不为空.让URLStreamHandlerFactory生成
3. 根据协议 `protocol`生成,注意老版本中`http和https`用的是

    streamHandler = new HttpHandler();  
    streamHandler = new HttpsHandler();

后来的版本用的是,底层换成了okhttp的类

	handler = (URLStreamHandler)Class.forName("com.android.okhttp.HttpHandler").newInstance();

	handler = (URLStreamHandler)Class.forName("com.android.okhttp.HttpsHandler").newInstance();

最后还有个并发检查,避免因为多线程的原因导致生成多个`handler`实例

	 synchronized (streamHandlerLock) {

        URLStreamHandler handler2 = null;

        // Check again with hashtable just in case another
        // thread created a handler since we last checked
        handler2 = handlers.get(protocol);

        if (handler2 != null) {
            return handler2;
        }

        // Check with factory if another thread set a
        // factory since our last check
        if (!checkedWithFactory && factory != null) {
            handler2 = factory.createURLStreamHandler(protocol);
        }

        if (handler2 != null) {
            // The handler from the factory must be given more
            // importance. Discard the default handler that
            // this thread created.
            handler = handler2;
        }

        // Insert this handler into the hashtable
        if (handler != null) {
            handlers.put(protocol, handler);
        }

    }

URLStreamHandler是一个抽象类,子类包括http,https,ftp等实现类，这里我们只看http的实现类

	protected URLConnection openConnection(URL var1) throws IOException {
        return this.openConnection(var1, (Proxy)null);
    }

    protected URLConnection openConnection(URL var1, Proxy var2) throws IOException {
        return new HttpURLConnection(var1, var2, this);
    }

这里会生成一个HttpUrlConnection()对象，注意这里的HttpUrlConnection()对象和最开始的HttpUrlConnection()对象不是一个，包名不同,一个是sun公司的包，一个是谷歌官方的包。

**sun/HttpURLConnection.java**

	protected HttpURLConnection(URL var1, Proxy var2, Handler var3) {
        super(var1);
        ....
        if(this.instProxy instanceof ApplicationProxy) {
            try {
                this.cookieHandler = CookieHandler.getDefault();
            } catch (SecurityException var5) {
                ;
            }
        } else {
            this.cookieHandler = (CookieHandler)AccessController.doPrivileged(new PrivilegedAction() {
                public CookieHandler run() {
                    return CookieHandler.getDefault();
                }
            });
        }

        this.cacheHandler = (ResponseCache)AccessController.doPrivileged(new PrivilegedAction() {
            public ResponseCache run() {
                return ResponseCache.getDefault();
            }
        });
    }

去掉部分不重要的代码之后,剩下的代码貌似都在获取缓存值。没有进行正常的网络连接。

**此时猜测** API26的HttpUrlConnection的网络请求发生在 `getInputStream()`中,而 `openConnection()`是为获取上次请求的缓存状态

### getInputStream() ###

**URLConnection.java中**

	public InputStream getInputStream() throws IOException {
        throw new UnknownServiceException("protocol doesn't support input");
    }

同样`URLConnection`是个抽象类,跳到对应的HttpUrlConnection.java类中

	public synchronized InputStream getInputStream() throws IOException {
        this.connecting = true;
        SocketPermission var1 = this.URLtoSocketPermission(this.url);
        if(var1 != null) {
            try {
                return (InputStream)AccessController.doPrivilegedWithCombiner(new PrivilegedExceptionAction() {
                    public InputStream run() throws IOException {
                        return HttpURLConnection.this.getInputStream0();
                    }
                }, (AccessControlContext)null, new Permission[]{var1});
            } catch (PrivilegedActionException var3) {
                throw (IOException)var3.getException();
            }
        } else {
            return this.getInputStream0();
        }
    }
一系列处理最后都会跳转到`getInputStream0()`函数内

getInputStream0()函数中的代码过多，就不贴出来了。来大概讲下流程
	
	//判断是否有缓存，有的话直接返回
	if(this.inputStream != null) {
       return this.inputStream;
	}

	
	//是否正在进行输入输出字符流的行为，如果有的判断输出流,因为这时网络连接还未建立,所以只可能是输出流到缓存
     if(this.streaming()) {
        if(this.strOutputStream == null) {
            this.getOutputStream();
        }

        this.strOutputStream.close();
        if(!this.strOutputStream.writtenOK()) {
            throw new IOException("Incomplete output stream");
        }
      }

同样 getOutputStream 会跳到getOutputStream0()中

**getOutputStream()**
		
	//会对URL做一次校验,主要针对host,header,protocol和authority等
	SocketPermission var1 = this.URLtoSocketPermission(this.url);

**getOutputStream0()**
	
	//有写流的行为的话,就一定是POST请求,因为GET请求不需要请求体
	//所以强制修改method为POST
 	if(this.method.equals("GET")) {
        this.method = "POST";
    }
	
	//检查是否有可用的链接,没有则进行重新链接
	if(!this.checkReuseConnection()) {
        this.connect();
    }

**checkReuseConnection()**
	

	//先检查connected,再检查reuseClient,说明reuseClient是connect的基础连接
 	private boolean checkReuseConnection() {
        if(this.connected) {
            return true;
        } else if(this.reuseClient != null) {
            this.http = this.reuseClient;
            this.http.setReadTimeout(this.getReadTimeout());
            this.http.reuse = false;
            this.reuseClient = null;
            this.connected = true;
            return true;
        } else {
            return false;
        }
    }

回到**getOutputStream0(**)

	//添加请求部分
	if(this.streaming() && this.strOutputStream == null) {
        this.writeRequests();
    }

接下来回到主流程 connect()部分会调用 `plainConnect0()`方法
	
有缓存拿缓存，缓存信息完整的话就不需要重新进行网络连接了

 	if(this.cacheHandler != null && this.getUseCaches()) {
        try {
			//对url做一次处理,兼容一些缺少/等情况的url
            URI var1 = ParseUtil.toURI(this.url);
            if(var1 != null) {
                this.cachedResponse = this.cacheHandler.get(var1, this.getRequestMethod(), this.getUserSetHeaders().getHeaders());
                if("https".equalsIgnoreCase(var1.getScheme()) && !(this.cachedResponse instanceof SecureCacheResponse)) {
                    this.cachedResponse = null;
                }

                if(logger.isLoggable(Level.FINEST)) {
                    logger.finest("Cache Request for " + var1 + " / " + this.getRequestMethod());
                    logger.finest("From cache: " + (this.cachedResponse != null?this.cachedResponse.toString():"null"));
                }

                if(this.cachedResponse != null) {
                    this.cachedHeaders = this.mapToMessageHeader(this.cachedResponse.getHeaders());
                    this.cachedInputStream = this.cachedResponse.getBody();
                }
            }
        } catch (IOException var6) {
            ;
        }

        if(this.cachedHeaders != null && this.cachedInputStream != null) {
            this.connected = true;
            return;
        }

        this.cachedResponse = null;
    }

没有连接缓存的话
	
	//第一次连接还是再次连接对应的方法不一样
  	if(!this.failedOnce) {
	    this.http = this.getNewHttpClient(this.url, (Proxy)null, this.connectTimeout);
	    this.http.setReadTimeout(this.readTimeout);
	} else {
	    this.http = this.getNewHttpClient(this.url, (Proxy)null, this.connectTimeout, false);
	    this.http.setReadTimeout(this.readTimeout);
	}

不管是否是第一次连接都会生成一个HttpClient对象,而这个对象才是连接网络的主要成员。其中变量var3表示是否是第一次发起连接，如果是,并且httpClient对象var5是有缓存的,这时需要做一些缓存清理，变量重置的操作。因为需要重新开启一个新的连接，需要先把老的连接清理掉


 	public static HttpClient New(URL var0, Proxy var1, int var2, boolean var3, HttpURLConnection var4) throws IOException {
        if(var1 == null) {
            var1 = Proxy.NO_PROXY;
        }

        HttpClient var5 = null;
        if(var3) {
            var5 = kac.get(var0, (Object)null);
            if(var5 != null && var4 != null && var4.streaming() && var4.getRequestMethod() == "POST" && !var5.available()) {
                var5.inCache = false;
                var5.closeServer();
                var5 = null;
            }

            if(var5 != null) {
                if(var5.proxy != null && var5.proxy.equals(var1) || var5.proxy == null && var1 == null) {
                    synchronized(var5) {
                        var5.cachedHttpClient = true;

                        assert var5.inCache;

                        var5.inCache = false;
                        if(var4 != null && var5.needsTunneling()) {
                            var4.setTunnelState(TunnelState.TUNNELING);
                        }

                        logFinest("KeepAlive stream retrieved from the cache, " + var5);
                    }
                } else {
                    synchronized(var5) {
                        var5.inCache = false;
                        var5.closeServer();
                    }

                    var5 = null;
                }
            }
        }
		
		//生成HttpClient对象，进行网络连接
        if(var5 == null) {
            var5 = new HttpClient(var0, var1, var2);
        } else {
            SecurityManager var6 = System.getSecurityManager();
            if(var6 != null) {
                if(var5.proxy != Proxy.NO_PROXY && var5.proxy != null) {
                    var6.checkConnect(var0.getHost(), var0.getPort());
                } else {
                    var6.checkConnect(InetAddress.getByName(var0.getHost()).getHostAddress(), var0.getPort());
                }
            }

            var5.url = var0;
        }

        return var5;
    }

**HttpClient.java **

    protected HttpClient(URL var1, Proxy var2, int var3) throws IOException {
        this.cachedHttpClient = false;
        this.poster = null;
        this.failedOnce = false;
        this.ignoreContinue = true;
        this.usingProxy = false;
        this.keepingAlive = false;
        this.keepAliveConnections = -1;
        this.keepAliveTimeout = 0;
        this.cacheRequest = null;
        this.reuse = false;
        this.capture = null;
        this.proxy = var2 == null?Proxy.NO_PROXY:var2;
        this.host = var1.getHost();
        this.url = var1;
        this.port = var1.getPort();
        if(this.port == -1) {
            this.port = this.getDefaultPort();
        }

        this.setConnectTimeout(var3);
        this.capture = HttpCapture.getCapture(var1);
		//开启连接服务
        this.openServer();
    }

openServer()函数内调用 openServer(this.host, this.port);
传入host和port开启 serverSocket.setTcpNoDelay(true);服务

网络连接成功,http请求参数也已经设置成功，接下来只等待网络回调把数据流写会缓存即可。

### 补充说明 ###

- HttpURLConnection的connect()函数，实际上只是建立了一个与服务器的tcp连接，并没有实际发送http请求。 无论是post还是get，http请求实际上直到HttpURLConnection的getInputStream()这个函数里面才正式发送出去。
- 在用POST方式发送URL请求时，URL请求参数的设定顺序是重中之重， 对connection对象的一切配置（那一堆set函数） 都必须要在connect()函数执行之前完成。而对outputStream的写操作，又必须要在inputStream的读操作之前。 这些顺序实际上是由http请求的格式决定的。
- http请求实际上由两部分组成， 一个是http头，所有关于此次http请求的配置都在http头里面定义， 一个是正文content。 connect()函数会根据HttpURLConnection对象的配置值生成http头部信息，因此在调用connect函数之前， 就必须把所有的配置准备好。
- 在http头后面紧跟着的是http请求的正文，正文的内容是通过outputStream流写入的，实际上outputStream不是一个网络流，充其量是个字符串流，往里面写入的东西不会立即发送到网络， 而是存在于内存缓冲区中，待outputStream流关闭时，根据输入的内容生成http正文。 至此，http请求的东西已经全部准备就绪。在getInputStream()函数调用的时候，就会把准备好的http请求 正式发送到服务器了，然后返回一个输入流，用于读取服务器对于此次http请求的返回信息。由于http 请求在getInputStream的时候已经发送出去了（包括http头和正文），因此在getInputStream()函数 之后对connection对象进行设置（对http头的信息进行修改）或者写入outputStream（对正文进行修改） 都是没有意义的了，执行这些操作会导致异常的发生。

### 参考文章 ###

[Android-浅析-HttpURLConnection](https://link.jianshu.com/?t=https://jasonzhong.github.io/2017/01/26/Android-浅析-HttpURLConnection/)  
[网络请求HttpURLConnection剖析](https://blog.csdn.net/linglongxin24/article/details/52881950)  
[Android每周一轮子：HttpURLConnection](https://juejin.im/post/5aa51b0e518825558c470d00)

### 另外 ###

[个人的github](https://github.com/niknowzcd/AndroidNote)  
[闲暇之余写的故事](https://book.qidian.com/info/1011888583)



[](http://fucknmb.com/2017/07/13/Android%E4%BB%A3%E7%90%86%E7%B3%BB%E7%BB%9F%E7%BA%A7HttpUrlConnection%E8%AF%B7%E6%B1%82%E5%88%B0%E7%AC%AC%E4%B8%89%E6%96%B9%E7%BD%91%E7%BB%9C%E5%BA%93-%E6%94%AF%E6%8C%81Http-2-0%E5%8F%8AHttpDNS/#more)




