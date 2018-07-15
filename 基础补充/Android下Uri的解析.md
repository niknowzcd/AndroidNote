
### URL,URN,URI的关系 ###

URI在于I(**Identifier**)是统一资源标识符,可以唯一标识一个资源。  
URL在于L(**Location**)是统一资源定位符,可以提供找到该资源的路径。  
URN在于N(**name**)统一资源名称,通过名字标识资源。


一般来说URL和URN都是URI的子集,它们两者共同组成了URI。

比如[https://www.zhihu.com/question/21950864](https://www.zhihu.com/question/21950864),这一串地址可以唯一标识一个资源,所以是一个URI,同时也可以通过这个地址找到资源,所以也是一个URL。

还有一种情况,**urn:isbn:0-486-27557-4**这是一本书的isbn,可以唯一标识一本书,但我们无法通过这一串isbn找到书籍,所以这里是URI,不是URL,准确说这一串isbn是一个URN。

### URI的结构 ###

URI的几种划分形式


> 基本划分

	[scheme:]scheme-specific-part[#fragment] 

> 进一步划分

    [scheme:][//authority][path][?query][#fragment]

> 再进一步划分

	[scheme:][//host:port][path][?query][#fragment] 

**其中有一些简单的规则**

- path可以有多个，每个用/连接，比如
  `scheme://authority/path1/path2/path3?query#fragment`
- query参数可以带有对应的值，也可以不带，如果带对应的值用=表示，如:
  `scheme://authority/path1/path2/path3?id = 1#fragment`，这里有一个参数id，它的值是1
- query参数可以有多个，每个用&连接
  `scheme://authority/path1/path2/path3?id = 1&name = mingming&old#fragment ` 
  这里有三个参数：  
  参数1：id，其值是:1  
  参数2：name，其值是:mingming  
  参数3：old，没有对它赋值，所以它的值是null
- 在android中，除了scheme、authority是必须要有的，其它的几个path、query、fragment，它们每一个可以选择性的要或不要，但顺序不能变，比如：  
  其中"path"可不要：scheme://authority?query#fragment
  其中"path"和"query"可都不要：scheme://authority#fragment
  其中"query"和"fragment"可都不要：scheme://authority/path
  "path","query","fragment"都不要：scheme://authority

### 做一个简单的例子匹配 ###

	http://www.java2s.com:8080/yourpath/fileName.htm?name=张三&id=4#niknowzcd  

- scheme:http
- host:www.java2s.com
- port:8080
- path:/yourpath/fileName.htm
- query:name=张三&id=4
- fragment:niknowzcd  

### 常用的api ###

同样的例子

	http://www.java2s.com:8080/yourpath/fileName.htm?name=张三&id=4#niknowzcd  

- **getScheme() :**获取Uri中的scheme字符串部分，在这里即**http**
- **getSchemeSpecificPart():**获取Uri中的scheme-specific-part:部分，这里是：**//www.java2s.com:8080/yourpath/fileName.htm?**
- **getFragment():**获取Uri中的Fragment部分，**niknowzcd**
- **getAuthority():**获取Uri中Authority部分，即**www.java2s.com:8080**
- **getPath():**获取Uri中path部分，即**/yourpath/fileName.htm**
- **getQuery():**获取Uri中的query部分，即**name=张三&id=4**
- **getHost():**获取Authority中的Host字符串，即**www.java2s.com**
- **getPost():**获取Authority中的Port字符串，即**8080**

另外还有两个常用的属性:`getPathSegments()`，`getQueryParameter(String key)`

> List<String> getPathSegments() 会将整个path路径保存下来,并以/符号作为分隔符

	String mUriStr = "http://www.java2s.com:8080/yourpath/fileName.htm?stove=10&path=32&id=4#harvic";  
	Uri mUri = Uri.parse(mUriStr);  
	List<String> pathSegList = mUri.getPathSegments();  
	for (String pathItem:pathSegList){  
	    Log.d("debug",pathItem);  
	}  
	
	//输出
	yourpath	
	fileName.htm

> getQueryParameter(String key): 则是根据key来获取对应的Query值

	String mUriStr = "http://www.java2s.com:8080/yourpath/fileName.htm?name=张三&id=4#niknowzcd";  
	mUri = Uri.parse(mUriStr);  
	Log.d(debug,"getQueryParameter(\"name\"):"+mUri.getQueryParameter("name"));  
	Log.d(debug,"getQueryParameter(\"id\"):"+mUri.getQueryParameter("id"));  

	//输出
	getQueryParameter("name"):张三
	getQueryParameter("id"):

**在path中，即使针对某一个KEY不对它赋值是允许的，但在利用getQueryParameter()获取该KEY对应的值时，获取到的是null;**

### 匹配URI跳转 ###




