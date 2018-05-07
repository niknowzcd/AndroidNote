## WebView##
![](http://upload-images.jianshu.io/upload_images/944365-e8e5a75c56f059a8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### webSettings (配置属性)###
### WebViewClient (处理通知&请求事件) ###
### WebChromeClient (处理对话框，网站图标，title，加载进度) ###


### WebView的状态 ###
	
	//激活WebView为活跃状态，能正常执行网页的响应
	webView.onResume() ；
	
	//当页面被失去焦点被切换到后台不可见状态，需要执行onPause
	//通过onPause动作通知内核暂停所有的动作，比如DOM的解析、plugin的执行、JavaScript执行。
	webView.onPause()；
	
	//当应用程序(存在webview)被切换到后台时，这个方法不仅仅针对当前的webview而是全局的全应用程序的webview
	//它会暂停所有webview的layout，parsing，javascripttimer。降低CPU功耗。
	webView.pauseTimers()
	//恢复pauseTimers状态
	webView.resumeTimers()；
	
	//销毁Webview
	//在关闭了Activity时，如果Webview的音乐或视频，还在播放。就必须销毁Webview
	//但是注意：webview调用destory时,webview仍绑定在Activity上
	//这是由于xml中定义webview构建时传入了该Activity的context对象
	//因此需要先从父容器中移除webview,然后再销毁webview:
	rootLayout.removeView(webView); 
	webView.destroy();
	
	//动态创建WebView,传入getApplicationContext(),这样就不会引起内存泄漏了

### 网页前进后退 ###

	//是否可以后退
	Webview.canGoBack() 
	//后退网页
	Webview.goBack()
	
	//是否可以前进                     
	Webview.canGoForward()
	//前进网页
	Webview.goForward()
	
	//以当前的index为起始点前进或者后退到历史记录中指定的steps
	//如果steps为负数则为后退，正数则为前进
	Webview.goBackOrForward(intsteps) 

**Back键控制网页后退/网页上的返回箭头也可以调用原生的方法实现网页后退的效果**

	public boolean onKeyDown(int keyCode, KeyEvent event) {
	    if ((keyCode == KEYCODE_BACK) && mWebView.canGoBack()) { 
	        mWebView.goBack();
	        return true;
	    }
	    return super.onKeyDown(keyCode, event);
	}


### 清除缓存数据 ###

	//清除网页访问留下的缓存
	//由于内核缓存是全局的因此这个方法不仅仅针对webview而是针对整个应用程序.
	Webview.clearCache(true);
	
	//清除当前webview访问的历史记录
	//只会webview访问历史记录里的所有记录除了当前访问记录
	Webview.clearHistory()；
	
	//这个api仅仅清除自动完成填充的表单数据，并不会清除WebView存储到本地的数据
	Webview.clearFormData()；

## WebSettings类 ##

> 作用：对WebView进行配置和管理

### 常见方法 ###

	//声明WebSettings子类
	WebSettings webSettings = webView.getSettings();
	
	//如果访问的页面中要与Javascript交互，则webview必须设置支持Javascript
	webSettings.setJavaScriptEnabled(true);  
	
	//支持插件
	webSettings.setPluginsEnabled(true); 
	
	//设置自适应屏幕，两者合用
	webSettings.setUseWideViewPort(true); //将图片调整到适合webview的大小 
	webSettings.setLoadWithOverviewMode(true); // 缩放至屏幕的大小
	
	//缩放操作
	webSettings.setSupportZoom(true); //支持缩放，默认为true。是下面那个的前提。
	webSettings.setBuiltInZoomControls(true); //设置内置的缩放控件。若为false，则该WebView不可缩放
	webSettings.setDisplayZoomControls(false); //隐藏原生的缩放控件
	
	//其他细节操作
	webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK); //关闭webview中缓存 
	webSettings.setAllowFileAccess(true); //设置可以访问文件 
	webSettings.setJavaScriptCanOpenWindowsAutomatically(true); //支持通过JS打开新窗口 
	webSettings.setLoadsImagesAutomatically(true); //支持自动加载图片
	webSettings.setDefaultTextEncodingName("utf-8");//设置编码格式


### 常见用法 ###
- 当加载 html 页面时，WebView会在/data/data/包名目录下生成 database 与 cache 两个文件夹
- 请求的 URL记录保存在 WebViewCache.db，而 URL的内容是保存在 WebViewCache 文件夹下


        mWebSettings = mWebView.getSettings();
        //优先使用缓存:
        mWebSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);
        //缓存模式如下：
        //LOAD_CACHE_ONLY: 不使用网络，只读取本地缓存数据
        //LOAD_DEFAULT: （默认）根据cache-control决定是否从网络上取数据。
        //LOAD_NO_CACHE: 不使用缓存，只从网络获取数据.
        //LOAD_CACHE_ELSE_NETWORK，只要本地有，无论是否过期，或者no-cache，都使用缓存中的数据。

        //不使用缓存:
        mWebSettings.setCacheMode(WebSettings.LOAD_NO_CACHE);

- 离线加载


        if (NetStatusUtil.isConnected(getApplicationContext())) {
            webSettings.setCacheMode(WebSettings.LOAD_DEFAULT);//根据cache-control决定是否从网络上取数据。
        } else {
            webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);//没网，则从本地获取，即离线加载
        }

        webSettings.setDomStorageEnabled(true); // 开启 DOM storage API 功能
        webSettings.setDatabaseEnabled(true);   //开启 database storage API 功能
        webSettings.setAppCacheEnabled(true);//开启 Application Caches 功能

        String cacheDirPath = getFilesDir().getAbsolutePath() + APP_CACAHE_DIRNAME;
        webSettings.setAppCachePath(cacheDirPath); //设置  Application Caches 缓存目录

**每个 Application 只调用一次 WebSettings.setAppCachePath()，WebSettings.setAppCacheMaxSize()**

### WebViewClient ###
- 作用：处理各种通知 & 请求事件

**常见方法1.shouldOverrideUrlLoading()**

        //方式1. 加载一个网页：
        mWebView.loadUrl("http://www.google.com/");

        //方式2：加载apk包中的html页面
        mWebView.loadUrl("file:///android_asset/test.html");

        //方式3：加载手机本地的html页面
        mWebView.loadUrl("content://com.android.htmlfileprovider/sdcard/test.html");

        //步骤3. 复写shouldOverrideUrlLoading()方法，使得打开网页时不调用系统浏览器， 而是在本WebView中显示
        mWebView.setWebViewClient(new WebViewClient(){
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }
        });

**其他常见方法**
	
	//开始载入页面调用，可以设置一个loading
	onPageStarted();  
	//页面载入结束，关闭loading
	onPageFinished();
	//加载页面资源时调用，每一个资源(比如图片)都会有回调
	onLoadResource();
	//加载页面的服务器出现错误时回调(如404)
	onReceivedError();

	//示例
    mWebView.setWebViewClient(new WebViewClient(){
        @Override
        public void onReceivedError(WebView view, int errorCode, String description, String failingUrl){
            switch(errorCode)
            {
                case 404:
                    view.loadUrl("file:///android_assets/error_handle.html");
                    break;
            }
        }
    });

    //onReceivedSslError() 处理https请求
    //webView默认是不处理https请求的，页面显示空白，需要进行如下设置：
    mWebView.setWebViewClient(new WebViewClient() {
        @Override
        public void onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) {
            handler.proceed();    //表示等待证书响应
            // handler.cancel();      //表示挂起连接，为默认方式
            // handler.handleMessage(null);    //可做其他处理
        }
    });

### WebChromeClient类 ###
> 辅助 WebView 处理 Javascript 的对话框,网站图标,网站标题等等。

    mWebView.setWebChromeClient(new WebChromeClient(){
		//页面加载进度
        @Override
        public void onProgressChanged(WebView view, int newProgress) {
            super.onProgressChanged(view, newProgress);
        }	
		//网页标题
        @Override
        public void onReceivedTitle(WebView view, String title) {
            super.onReceivedTitle(view, title);
        }

    });


## 注意事项 ##
### 避免WebView内存泄漏 ###
1.不在xml中定义WebView，因为在xml中传递进去的会是当前Activity的引用。在需要的时候，动态创建

	LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
	        mWebView = new WebView(getApplicationContext());
	        mWebView.setLayoutParams(params);
	        mLayout.addView(mWebView);

2.Activity销毁WebView,因为有的时候webView里面异步播放多媒体文件，会导致webView无法及时销毁

	@Override
    protected void onDestroy() {
        if (mWebView != null) {
            mWebView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null);
            mWebView.clearHistory();

            ((ViewGroup) mWebView.getParent()).removeView(mWebView);
            mWebView.destroy();
            mWebView = null;
        }
        super.onDestroy();
    }	
		

## Android与JS交互 ##

![](http://upload-images.jianshu.io/upload_images/944365-29c6a46c81304f4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> Android与js的交互的桥梁是WebView

**Android调用js**  
1. webView的loadUrl()  
2. webView的evaluateJavascript()

**js调用Android**  
1. 通过WebView的addJavascriptInterface()进行对象映射  
2. 通过WebViewClient的shouldOverrideUrlLoading()方法回调拦截URL  
3. 通过WebChromeClient的onJsAlert(),onJsConfirm(),onJsPrompt()方法回调拦截JS对话框alert()、confirm()、prompt()消息


**Android调用JS**

	// Android版本变量
	final int version = Build.VERSION.SDK_INT;
	// 因为该方法在 Android 4.4 版本才可使用，所以使用时需进行版本判断
	if (version < 18) {
	    mWebView.loadUrl("javascript:callJS()");
	} else {
	    mWebView.evaluateJavascript（"javascript:callJS()", new ValueCallback<String>() {
	        @Override
	        public void onReceiveValue(String value) {
	            //此处为 js 返回的结果
	        }
	    });
	}
调用JS的方法通用的格式 `javascript:methodName()`
一般在4.4的版本之后都用后者，效率高，调用方便


### JS调用Android ###
> 通过WebView的addJavascriptInterface()进行对象映射  
	
	//这个方法的两个参数实际上是映射关系。第一个参数表示Android原生的对象，第二个参数是提供给JS使用的。
	mWebView.addJavascriptInterface(new AndroidLocalMethod(), "JsObject");	

定义Android原生对象,在这里定义方法体,供JS调用
	
	public class AndroidLocalMethod {
	    // 被JS调用的方法必须加入@JavascriptInterface注解

	    @JavascriptInterface
	    public void hello(String msg) {
	        System.out.println("JS调用了Android的hello方法");
	    }
	
	    @JavascriptInterface
	    public void hello2(String msg) {
	        System.out.println(msg);
	    }

	    @JavascriptInterface
	    public void hello3(String msg) {
	        System.out.println(msg);
	    }
	}

JS调用
	 
	//因为做了映射，所以在JS中JsObject就代表着Android对象AndroidLocalMethod
	//自然就可以随意调用该类下的所有方法了
     function callAndroid(){
        JsObject.hello("js调用了android中的hello方法");
		JsObject.hello2("js调用了android中的hello方法");
		JsObject.hello3("js调用了android中的hello方法");
     }

**优点**:调用方便  
**缺点**:容易被攻击.因为在4.2版本之前，JS方可以任意调用Android端的所有方法，从而获取一些用户信息。不过在4.2版本之后加入了`@JavascriptInterface`注解之后，就可以不必担心这个问题了


> 在Android通过WebViewClient复写shouldOverrideUrlLoading()

当`WebView.loadUrl()`之后，会回调`WebViewClient类的shouldOverrideUrlLoading`，这个时候可以根据回传的URL做重定向操作

	//1.webView.loadUrl()
	//2.重写WebViewClient类的shouldOverrideUrlLoading
	//3.JS端重定义URL
    function callAndroid(){
       	document.location = "js://webview?arg1=111&arg2=222";
 	}
	//在shouldOverrideUrlLoading重写相关逻辑

> 通过WebChromeClient的onJsAlert(),onJsConfirm(),onJsPrompt()方法回调拦截JS对话框alert()、confirm()、prompt()消息

**Android通过 WebChromeClient 的onJsAlert()、onJsConfirm()、onJsPrompt（）方法回调分别拦截JS对话框 
（即上述三个方法），得到他们的消息内容，然后解析即可**

**比较常用的是`prompt()`方法,因为只有prompt()可以返回任意类型的值**


[WebView详解](http://blog.csdn.net/carson_ho/article/details/52693322)  
[WebView与JS之间的交互](http://blog.csdn.net/carson_ho/article/details/64904691)  
[WebView的使用漏洞](http://blog.csdn.net/carson_ho/article/details/64904635)  
[WebView的缓存机制和资源预加载方案](http://blog.csdn.net/carson_ho/article/details/71402764)