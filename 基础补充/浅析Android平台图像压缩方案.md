
在介绍Android平台的压缩方案之前,先了解一下Bitmap的几个主要概念。

> 像素密度

像素密度指的是每英寸像素数目，在Bitmap里用mDensity/mTargetDensity，mDensity默认是设备屏幕的像素密度，mTargetDensity是图片的目标像素密度，在加载图片时就是 drawable 目录的像素密度。

> 色彩模式->色彩模式是数字世界中表示颜色的一种算法，在Bitmap里用Config来表示。

- ARGB_8888：每个像素占四个字节，A、R、G、B 分量各占8位，是 Android 的默认设置；
- RGB_565：每个像素占两个字节，R分量占5位，G分量占6位，B分量占5位；
- ARGB_4444：每个像素占两个字节，A、R、G、B分量各占4位，成像效果比较差；
- Alpha_8: 只保存透明度，共8位，1字节；

> Bitmap的计算方式

    memory=scaledWidth*scaledHeight*每个像素所占字节数

其中  
scaledWidth :  width*targetDensity/density+0.5  
scaledHeight： height*targetDensity/density+0.5

- `scaledWidth`表示水平方向的像素值,  
- `width`表示屏幕宽度,  
- `targetDensity`表示手机的像素密度,这个值一般跟手机相关,  
- `density`表示decodingBitmap 的 density,这个值一般跟图片放置的目录有关(hdpi/xxhdpi)

scaledHeight同理

**每个像素所占字节数**:这个值跟色彩模式相关，默认 ARGB_8888 则是4个字节，

在Bitmap种有两个获取内存占用大小的方法

- getByteCount()：API12 加入，代表存储 Bitmap 的像素需要的最少内存。
- getAllocationByteCount()：API19 加入，代表在内存中为 Bitmap 分配的内存大小，代替了 getByteCount() 方法。

> 两者的区别:

在不复用 Bitmap 时，getByteCount() 和 getAllocationByteCount 返回的结果是一样的。在通过复用 Bitmap 来解码图片时，那么 getByteCount() 表示新解码图片占用内存的大小，getAllocationByteCount() 表示被复用 Bitmap真实占用的内存大小（即 mBuffer 的长度）。


## 图片压缩方式 ##

### 质量压缩 ###

> 质量压缩的关键在于Bitmap.compress()函数，该函数不会改变图像的大小，但是可以降低图像的质量，从而降低存储大小，进而达到压缩的目的。

**这里提到的图像的质量主要指的是图片的色彩空间**

一般图像的色彩空间为**RGB**,主要通过RGB三原色通道来描述图片,其中又有**ARGB**格式,比起RGB多了一个透明度的通道。

Android下的质量压缩主要通过下面这个函数来实现的。
	
	bitmap.compress(Bitmap.CompressFormat.JPEG, 100, outputStream);

三个参数

- CompressFormat format:压缩格式,它有JPEG、PNG、WEBP三种选择，JPEG是有损压缩，PNG是无损压缩，WEBP是Google推出的图像格式.
- int quality：0~100可选，数值越大，质量越高，图像越大。
- OutputStream stream：压缩后图像的输出流。

其中PNG是无损格式的,压缩效果不太理想,而WEBP会存在兼容性的问题。出于兼容性和效果来看，一般会选择JPEG作为压碎格式。

**实例代码**
	
	// R.drawable.thumb 为 png 图片
	Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.thumb);
	try {
	    //保存压缩图片到本地
	    File file = new File(Environment.getExternalStorageDirectory(), "aaa.jpg");
	    if (!file.exists()) {
	        file.createNewFile();
	    }
	    FileOutputStream fs = new FileOutputStream(file);
	    bitmap.compress(Bitmap.CompressFormat.JPEG, 50, fs);
	    Log.i(TAG, "onCreate: file.length " + file.length());
	    fs.flush();
	    fs.close();
	} catch (FileNotFoundException e) {
	    e.printStackTrace();
	} catch (IOException e) {
	    e.printStackTrace();
	}
	//查看压缩之后的 Bitmap 大小
	ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
	bitmap.compress(Bitmap.CompressFormat.JPEG, 50, outputStream);
	byte[] bytes = outputStream.toByteArray();
	Bitmap compress = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
	Log.i(TAG, "onCreate: bitmap.size = " + bitmap.getByteCount() + "   compress.size = " + compress.getByteCount());

我们再来看看`quality`参数被设置为50前后,两张图片的对比.  
**压缩前的图片**  
![](http://www.52im.net/data/attachment/forum/201711/14/124552zwnonwd0200dxjgu.jpg)

**压缩后的图片**  
![](http://www.52im.net/data/attachment/forum/201711/14/124552w5duhwwqdutpbdnn.jpg)

从上述两图可以明显图片质量的差别,另外再通过log打印查看会压缩前后图片的所占用的大小是一样的。

即

	bitmap.size = compress.size
	
**Q:**这里可能有人就会有疑惑,为什么压缩过后,两张图片的大小还会是一样的呢？

**A:**因为图片在内存中的存储方式和文件中的存储方式是不一样的。图片压缩只会影响文件的大小,在这个例子中,压缩过后存到磁盘的文件大小会比压缩之前的文件大小减小很多。

内存中所占的大小没有变化是因为bitmap没有变化的原因。

**文章最开始提到Bitmap的计算方式**

	memory=scaledWidth*scaledHeight*每个像素所占字节数

因为是压缩的质量,所有宽高都不变,而每个像素所占的字节数跟色彩空间有关,默认是`ARGB_8888`.宽高不变,色彩空间不重新设置,那么bitmap所占的大小就不会发生改变。

说道这里可能又会有个新疑问  

**Q:**bitmap占用的大小不变,那为什么图片质量下降了呢？这是因为图片被压缩过了啊!

**A:**首先要知道JPEG格式是有损压缩的,JPEG格式的图片是不支持透明色彩的,这也是JPEG的大小会比PNG小很大,图片质量会比PNG差的原因。
在经过了`bitmap.compress()`这个流程时,JPEG会舍去透明属性.这样存放到磁盘时的文件大小就减小了.然后这个时候再通过`BitmapFactory.decodeByteArray()`把图片加载回来时,加载的是舍去了透明通道的图片,按理说应该采用 `RGB_565`或者`RGB_888`这样的色彩空间加载,但是你没有另外设置这个参数的话,加载的色彩格式会是默认`ARGB_8888`.图片都没有透明的色彩空间了,你再给它分配内存就只是浪费内存而已。

这也是为什么压缩前后,bitmap所占的大小相同,图片质量却有所差距的原因。

补充一个有趣的事件,在早期的Android平台下,对一张图片进行多次质量压缩,会得到一张变绿的图片。[详情链接](https://www.zhihu.com/question/29355920)


### 补充一些Android下各格式图片的存储方式 ###

> WebP

Webp图片格式是Google推出的一个支持alpha通道的有损压缩格式，据Google官方表明，同质量情况下Webp图像要比JPEG、PNG图像小25%~45%左右，在支持上Android4.0+版本提供原生支持，使用libwebp库进行编解码。

> GIF

GIF图像最广泛的应用是用于显示动画图像，它具备文件小且支持alpha通道的优点，不过它是由8位进行表示每个像素的色彩，仅支持256色，所以在对色彩要求比较高的场合不太适合。

> Stream

图片的存储形式从File转到内存中时，图片内容以字节方式存储在Stream中，此时所占的内存大小为File文件大小。

> Bitmap

在Android中，任何图片资源的显示对象都是通过bitmap来显示的，除了xml资源则是通过Canvas来绘制的，所以，对于某些纯色或者规则类的图像，可以通过xml进行描述或Canvas来绘制，这样所占用的内存比通过bitmap来显示将少几个等级。

> Bitmap与Drawable的联系

关于Bitmap和Drawable的关系，可以看官方的解释，Drawable是一个抽象的概念，来描述某些具备可绘制的的对象，它是一个抽象类，而Bitmap是一个最简单的Drawable实体对象，Bitmap并不继承于Drawable，它们之间建立关联最终是通过BitmapDrawable对象，该对象会把具体的Bitmap实例对象渲染到Canvas上。Drawable更注重描述的是某绘制的行为，而Bitmap则是注重存储着图像的像素信息。

> Bitmap存储空间

随着版本的变化以及存储空间的变化，Bitmap的存储空间主要有三个地方

**Native Memory**  
Android2.3以下版本，bitmap像素数据存储在native内存中，释放内存需主动调用recycle()方法  

**Dalvik Heap**  
Android3.0+版本，在Android2.3版本引入了并发的垃圾回收器后，在3.0以后的版本bitmap的像素数据则存储在虚拟机堆中，不需要主动调用recycle()来回收内存，gc会主动回收  

**Ashmem**  
匿名共享内存空间，说到这个，就会联想起大名鼎鼎的Fresco图片库，它巧妙的利用了这一空间来进行Bitmap对象的存储，对于Ashmem空间，首先想到的是与App进程空间是隔离且互不影响的，这点在Android4.4以下版本是这样的，在Android4.4+后版本，Ashmem空间将会包含在App所占用的内存空间中。看Fresco源码也可以看出，对于4.4+版本，对于Bitmap的解码使用了另外的解码器。在Android4.4以下版本如何使用Ashmem进行bitmap的存储呢？通过DecodeOptions：


	options.inPurgeable = true;
	options.inInputShareable = true;

以及通过MemoryFile可将图片的字节数据存储在Ashmem中。


## 尺寸压缩 ##

> 尺寸压缩本质上就是一个重新采样的过程，放大图像称为上采样，缩小图像称为下采样，Android提供了两种图像采样方法，邻近采样和双线性采样。

### 邻近采样 ###

> 邻近采样采用邻近点插值算法，用一个像素点代替邻近的像素点，

	BitmapFactory.Options options = new BitmapFactory.Options();
	options.inSampleSize = 2;
	Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
	Bitmap compress = BitmapFactory.decodeFile("/sdcard/test.png", options);

其中`options.inSampleSize`的值代表着压缩后一个像素点代替原来的几个像素点,比如`options.inSampleSize=2`,一个像素点会代替原来的2个像素点,注意这里的2个像素点仅仅指水平方向或者竖直方向上的。即原来2x2的像素,压缩后仅使用一个像素点来代替。

网上找了张图  

**压缩前的图片**

![](http://www.52im.net/data/attachment/forum/201711/15/101216n8xoqsdcxyy88dsq.png)

压缩后的图片

![](http://www.52im.net/data/attachment/forum/201711/15/101216um96ebsu4e5m9ueq.png)

压缩前红绿相间的图片,经过压缩后,完全变成了绿色.这时因为**邻近点插值算法**直接选择其中一个像素作为生成像素,另外一个像素直接抛弃,这样才会造成图片变成纯绿色的情况。

考虑到邻近采样的方法有些暴力,Android平台提供了另一种尺寸压缩方案

### 双线性采样 ###

> 双线性采样采用双线性插值算法，相比邻近采样简单粗暴的选择一个像素点代替其他像素点，双线性采样参考源像素相应位置周围2x2个点的值，根据相对位置取对应的权重，经过计算得到目标图像。

使用实例

	Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
	Bitmap compress = Bitmap.createScaledBitmap(bitmap, bitmap.getWidth()/2, bitmap.getHeight()/2, true);

或者

	Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
	Matrix matrix = new Matrix();
	matrix.setScale(0.5f, 0.5f);
	bm = Bitmap.createBitmap(bitmap, 0, 0, bit.getWidth(), bit.getHeight(), matrix, true);

**压缩效果**  

压缩前

![](http://www.52im.net/data/attachment/forum/201711/15/101401spfbbbzp2mqpubfp.png)

压缩后

![](http://www.52im.net/data/attachment/forum/201711/15/101401uficz2j2jejed0sv.png)

可以看出压缩后的图片不会像邻近采样那般只有纯粹的一种颜色,而是参考了像素源周围2x2个点的像素，并取其权重得到目标图像。

双线性采样相比邻近采样而言,图片的保真度会高些,但压缩的速率不及前者,因为前者不需要计算直接选择了其中一个像素作为生成像素。

### 双立方／双三次采样 (Android原生不支持) ###

> 双立方／双三次采样使用的是双立方／双三次插值算法。双立方／双三次插值算法参考了源像素某点周围 4x4 个像素。

双立方/双三次插值算法经常用于图像或者视频的缩放，它能比双线性内插值算法保留更好的细节质量。

双立方／双三次插值算法在平时的软件中是很常用的一种图片处理算法，但是这个算法有一个缺点就是计算量会相对比较大，是前三种算法中计算量最大的，软件 photoshop 中的图片缩放功能使用的就是这个算法。

### Lanczos 采样 (原生不支持)###

> Lanczos 采样和 Lanczos 过滤是 Lanczos 算法的两种常见应用，它可以用作低通滤波器或者用于平滑地在采样之间插入数字信号，Lanczos 采样一般用来增加数字信号的采样率，或者间隔采样来降低采样率。

### 采样效果 从低到高依次 ###

邻近采样--双线性采样--双立方／双三次采样--Lanczos 采样


[Android平台图像压缩方案](https://juejin.im/post/5a1bd6595188254cc067981f)  
[QQ音乐团队分享：Android中的图片压缩技术详解](http://www.52im.net/thread-1208-1-1.html)  
[也谈图片压缩](http://zhengxiaoyong.me/2017/04/23/%E4%B9%9F%E8%B0%88%E5%9B%BE%E7%89%87%E5%8E%8B%E7%BC%A9/)   
[为什么图片反复压缩后会普遍会变绿而不是其他颜色](https://www.zhihu.com/question/29355920)  
[Android之优雅地加载大图片](https://www.jianshu.com/p/0f56f35068e2)  
[内存占用/GPU渲染性能优化手记](https://wangfuda.github.io/2017/07/09/nebula_gpu_monitor_optimize/)

### 另外 ###

[个人的github](https://github.com/niknowzcd/AndroidNote)  
[闲暇之余写的故事](https://book.qidian.com/info/1011888583)


