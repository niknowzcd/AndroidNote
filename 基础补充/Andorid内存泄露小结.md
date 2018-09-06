### java内存的分配策略 ###
Java 程序运行时的内存分配策略有三种,分别是静态分配,栈式分配,和堆式分配，对应的，三种存储策略使用的内存空间主要分别是静态存储区（也称方法区）、栈区和堆区。

- 静态存储区（方法区）：主要存放静态数据、全局 static 数据和常量。这块内存在程序编译时就已经分配好，并且在程序整个运行期间都存在。
- 栈区 ：当方法被执行时，方法体内的局部变量（其中包括基础数据类型、对象的引用）都在栈上创建，并在方法执行结束时这些局部变量所持有的内存将会自动被释放。因为栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。
- 堆区 ： 又称动态内存分配，通常就是指在程序运行时直接 new 出来的内存，也就是对象的实例。这部分内存在不使用时将会由 Java 垃圾回收器来负责回收。

### 栈与堆的区别： ###
栈区:因为栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。所以栈区一般只给方法体内的变量分配内存空间，

堆区:空间容量足够，会给对象，及类中的变量开辟空间。


在方法体内定义的（局部变量）一些基本类型的变量和对象的引用变量都是在方法的栈内存中分配的。当在一段方法块中定义一个变量时，Java 就会在栈中为该变量分配内存空间，当超过该变量的作用域后，该变量也就无效了，分配给它的内存空间也将被释放掉，该内存空间可以被重新使用。

堆内存用来存放所有由 new 创建的对象（包括该对象其中的所有成员变量）和数组。在堆中分配的内存，将由 Java 垃圾回收器来自动管理。在堆中产生了一个数组或者对象后，还可以在栈中定义一个特殊的变量，这个变量的取值等于数组或者对象在堆内存中的首地址，这个特殊的变量就是我们上面说的引用变量。我们可以通过这个引用变量来访问堆中的对象或者数组。

实例:

    public class Sample {
	    int s1 = 0;
	    Sample mSample1 = new Sample();
	    
	    public void method() {
	    int s2 = 1;
	    Sample mSample2 = new Sample();
	    }
	}
    
    Sample mSample3 = new Sample();

Sample 类的局部变量 s2 和引用变量 mSample2 都是存在于栈中，但 mSample2 指向的对象是存在于堆上的。 mSample3 指向的对象实体存放在堆上，包括这个对象的所有成员变量 s1 和 mSample1，而它自己存在于栈中。

结论：

局部变量的基本数据类型和引用存储于栈中，引用的对象实体存储于堆中。—— 因为它们属于方法中的变量，生命周期随方法而结束。

成员变量全部存储与堆中（包括基本数据类型，引用和引用的对象实体）—— 因为它们属于类，类对象终究是要被new出来使用的。


### Java是如何管理内存 ###
Java的内存管理就是对象的分配和释放问题。在 Java 中，程序员需要通过关键字 new 为每个对象申请内存空间 (基本类型除外)，所有的对象都在堆 (Heap)中分配空间。另外，对象的释放是由 GC 决定和执行的。在 Java 中，内存的分配是由程序完成的，而内存的释放是由 GC 完成的，这种收支两条线的方法确实简化了程序员的工作。但同时，它也加重了JVM的工作。这也是 Java 程序运行速度较慢的原因之一。因为，GC 为了能够正确释放对象，GC 必须监控每一个对象的运行状态，包括对象的申请、引用、被引用、赋值等，GC 都需要进行监控。

监视对象状态是为了更加准确地、及时地释放对象，而释放对象的根本原则就是该对象不再被引用。

为了更好理解 GC 的工作原理，我们可以将对象考虑为有向图的顶点，将引用关系考虑为图的有向边，有向边从引用者指向被引对象。另外，每个线程对象可以作为一个图的起始顶点，例如大多程序从 main 进程开始执行，那么该图就是以 main 进程顶点开始的一棵根树。在这个有向图中，根顶点可达的对象都是有效对象，GC将不回收这些对象。如果某个对象 (连通子图)与这个根顶点不可达(注意，该图为有向图)，那么我们认为这个(这些)对象不再被引用，可以被 GC 回收。 以下，我们举一个例子说明如何用有向图表示内存管理。对于程序的每一个时刻，我们都有一个有向图表示JVM的内存分配情况。以下右图，就是左边程序运行到第6行的示意图。

![](https://camo.githubusercontent.com/ba01b8ae9af4a5e588251316c826bf3e0e695f35/687474703a2f2f7777772e69626d2e636f6d2f646576656c6f706572776f726b732f636e2f6a6176612f6c2d4a6176614d656d6f72794c65616b2f312e676966)

Java使用有向图的方式进行内存管理，可以消除引用循环的问题，例如有三个对象，相互引用，只要它们和根进程不可达的，那么GC也是可以回收它们的。这种方式的优点是管理内存的精度很高，但是效率较低。另外一种常用的内存管理技术是使用计数器，例如COM模型采用计数器方式管理构件，它与有向图相比，精度行低(很难处理循环引用的问题)，但执行效率很高。


### 什么是Java中的内存泄露 ###
在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是可达的，即在有向图中，存在通路可以与其相连；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

### Java内存泄漏引起的原因 ###
1.静态集合类引起内存泄漏：

像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放，因为他们也将一直被Vector等引用着。

    Static Vector v = new Vector(10);
    for (int i = 1; i<100; i++)
    {
    Object o = new Object();
    v.add(o);
    o = null;
    }


在这个例子中，循环申请Object 对象，并将所申请的对象放入一个Vector 中，如果仅仅释放引用本身（o=null），那么Vector 仍然引用该对象，所以这个对象对GC 来说是不可回收的。因此，如果对象加入到Vector 后，还必须从Vector 中删除，最简单的方法就是将Vector对象设置为null。


2、当集合里面的对象属性被修改后，再调用remove()方法时不起作用。

    public static void main(String[] args){
	    Set<Person> set = new HashSet<Person>();
	    Person p1 = new Person("唐僧","pwd1",25);
	    Person p2 = new Person("孙悟空","pwd2",26);
	    Person p3 = new Person("猪八戒","pwd3",27);
	    set.add(p1);
	    set.add(p2);
	    set.add(p3);
	    System.out.println("总共有:"+set.size()+" 个元素!"); //结果：总共有:3 个元素!
	    p3.setAge(2); //修改p3的年龄,此时p3元素对应的hashcode值发生改变
	    
	    set.remove(p3); //此时remove不掉，造成内存泄漏
	    
	    set.add(p3); //重新添加，居然添加成功
	    System.out.println("总共有:"+set.size()+" 个元素!"); //结果：总共有:4 个元素!
	    for (Person person : set)
	    {
	    	System.out.println(person);
	    }
    }

3.监听器
在java 编程中，我们都需要和监听器打交道，通常一个应用当中会用到很多监听器，我们会调用一个控件的诸如addXXXListener()等方法来增加监听器，但往往在释放对象的时候却没有记住去删除这些监听器，从而增加了内存泄漏的机会。

因为对象还和监听器保持联系，所有GC会认为这个对象还有用，不会去回收。

4、各种连接

比如数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，除非其显式的调用了其close（）方法将其连接关闭，否则是不会自动被GC 回收的。对于Resultset 和Statement 对象可以不进行显式回收，但Connection 一定要显式回收，因为Connection 在任何时候都无法自动回收，而Connection一旦回收，Resultset 和Statement 对象就会立即为NULL。但是如果使用连接池，情况就不一样了，除了要显式地关闭连接，还必须显式地关闭Resultset Statement 对象（关闭其中一个，另外一个也会关闭），否则就会造成大量的Statement 对象无法释放，从而引起内存泄漏。这种情况下一般都会在try里面去的连接，在finally里面释放连接。




## Android下常见的内存泄露 ##
### 1.集合类泄漏 ###

集合类如果仅仅有添加元素的方法，而没有相应的删除机制，导致内存被占用。如果这个集合类是全局性的变量 (比如类中的静态属性，全局性的 map 等即有静态引用或 final 一直指向它)，那么没有相应的删除机制，很可能导致集合所占用的内存只增不减。比如上面的典型例子就是其中一种情况，当然实际上我们在项目中肯定不会写这么 2B 的代码，但稍不注意还是很容易出现这种情况，比如我们都喜欢通过 HashMap 做一些缓存之类的事，这种情况就要多留一些心眼。

### 2.单例造成的内存泄漏 ###
	
	//这个时候传过来的context如果是activity的context就会造成内存泄露，因为单例是静态的，会导致对于的activity无法释放，
	
	//解决方法，传过来一个Appliaction的context，或者Application写一个静态方法 返回一个context
    public class AppManager {
	    private static AppManager instance;
	    private Context context;
	    private AppManager(Context context) {
	    	this.context = context;
	    }
	    public static AppManager getInstance(Context context) {
	    if (instance == null) {
	    	instance = new AppManager(context);
	    }
	    return instance;
	    }
    }

### 3.非静态内部类创建静态实例造成的内存泄漏 ###

### 4.匿名内部类 ###

android开发经常会继承实现Activity/Fragment/View，此时如果你使用了匿名类，并被异步线程持有了，那要小心了，如果没有任何措施这样一定会导致泄露

	public class MainActivity extends Activity {
    ...
    Runnable ref1 = new MyRunable();
    Runnable ref2 = new Runnable() {
        @Override
        public void run() {

        }
    };
       ...
    }

ref1和ref2的区别是，ref2使用了匿名内部类。我们来看看运行时这两个引用的内存：
![](https://camo.githubusercontent.com/2b1a52551d828d9640f23ee7c2802476b02ccec3/687474703a2f2f696d67322e746263646e2e636e2f4c312f3436312f312f666230356666366432653638663330396239346464383433353263383161636665306165383339653f73706d3d353137362e3130303233392e626c6f67636f6e742e31302e437455316334)

可以看到，ref1没什么特别的。

但ref2这个匿名类的实现对象里面多了一个引用：

this$0这个引用指向MainActivity.this，也就是说当前的MainActivity实例会被ref2持有，如果将这个引用再传入一个异步线程，此线程和此Acitivity生命周期不一致的时候，就造成了Activity的泄露。


### 5.Handler 造成的内存泄漏 ###

handler主要是为了处理耗时操作存在的，而且Handler,Message和MessageQueue是配套存在的。

当Handler发送的message没有被及时处理时(比如sleep)，线程MessageQueue会一直持有Message和发送它的Handler对象，同时也会引起对于的Activity无法及时释放，从而引起内存泄露。

**解决方法**:将Handler声明为静态内部类，这样Activity的生命周期就不会受到Activity的影响了。

推荐使用静态内部类 + WeakReference 这种方式。每次使用前注意判空。

![](https://camo.githubusercontent.com/068950c506eddc68677d6a84b5d7ffe1c03c6cf9/68747470733a2f2f67772e616c6963646e2e636f6d2f7470732f5442315536544e4c565858585863685846585858585858585858582d3634342d3534362e6a7067)

在Android应用的开发中，为了防止内存溢出，在处理一些占用内存大而且声明周期较长的对象时候，可以尽量应用软引用和弱引用技术。

软/弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。利用这个队列可以得知被回收的软/弱引用的对象列表，从而为缓冲器清除已失效的软/弱引用。

假设我们的应用会用到大量的默认图片，比如应用中有默认的头像，默认游戏图标等等，这些图片很多地方会用到。如果每次都去读取图片，由于读取文件需要硬件操作，速度较慢，会导致性能较低。所以我们考虑将图片缓存起来，需要的时候直接从内存中读取。但是，由于图片占用内存空间比较大，缓存很多图片需要很多的内存，就可能比较容易发生OutOfMemory异常。这时，我们可以考虑使用软/弱引用技术来避免这个问题发生。

使用软引用以后，在OutOfMemory异常发生之前，这些缓存的图片资源的内存空间可以被释放掉的，从而避免内存达到上限，避免Crash发生。

如果只是想避免OutOfMemory异常的发生，则可以使用软引用。如果对于应用的性能更在意，想尽快回收一些占用内存比较大的对象，则可以使用弱引用。

另外可以根据对象是否经常使用来判断选择软引用还是弱引用。如果该对象可能会经常使用的，就尽量用软引用。如果该对象不被使用的可能性更大些，就可以用弱引用。

ok，继续回到主题。前面所说的，创建一个静态Handler内部类，然后对 Handler 持有的对象使用弱引用，这样在回收时也可以回收 Handler 持有的对象，但是这样做虽然避免了 Activity 泄漏，不过 Looper 线程的消息队列中还是可能会有待处理的消息，所以我们在 Activity 的 Destroy 时或者 Stop 时应该移除消息队列 MessageQueue 中的消息。

下面几个方法都可以移除 Message：

    public final void removeCallbacks(Runnable r);
    
    public final void removeCallbacks(Runnable r, Object token);
    
    public final void removeCallbacksAndMessages(Object token);
    
    public final void removeMessages(int what);
    
    public final void removeMessages(int what, Object object);

###6.尽量避免使用 static 成员变量 ###
如果成员变量被声明为 static，那我们都知道其生命周期将与整个app进程生命周期一样。

这会导致一系列问题，如果你的app进程设计上是长驻内存的，那即使app切到后台，这部分内存也不会被释放。按照现在手机app内存管理机制，占内存较大的后台进程将优先回收，因为如果此app做过进程互保保活，那会造成app在后台频繁重启。当手机安装了你参与开发的app以后一夜时间手机被消耗空了电量、流量，你的app不得不被用户卸载或者静默。

这里修复的方法是：

不要在类初始时初始化静态成员。可以考虑lazy初始化。 架构设计上要思考是否真的有必要这样做，尽量避免。如果架构需要这么设计，那么此对象的生命周期你有责任管理起来。

### 7.避免 override finalize() --手动GC ###
1、finalize 方法被执行的时间不确定，不能依赖与它来释放紧缺的资源。时间不确定的原因是： 虚拟机调用GC的时间不确定 Finalize daemon线程被调度到的时间不确定

2、finalize 方法只会被执行一次，即使对象被复活，如果已经执行过了 finalize 方法，再次被 GC 时也不会再执行了，原因是：

含有 finalize 方法的 object 是在 new 的时候由虚拟机生成了一个 finalize reference 在来引用到该Object的，而在 finalize 方法执行的时候，该 object 所对应的 finalize Reference 会被释放掉，即使在这个时候把该 object 复活(即用强引用引用住该 object )，再第二次被 GC 的时候由于没有了 finalize reference 与之对应，所以 finalize 方法不会再执行。

3、含有Finalize方法的object需要至少经过两轮GC才有可能被释放。

### 8.资源未关闭造成的内存泄漏 ###
对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销，否则这些资源将不会被回收，造成内存泄漏。

### 9.一些不良代码造成的内存压力 ###
有些代码并不造成内存泄露，但是它们，或是对没使用的内存没进行有效及时的释放，或是没有有效的利用已有的对象而是频繁的申请新内存。

比如： Bitmap 没调用 recycle()方法，对于 Bitmap 对象在不使用时,我们应该先调用 recycle() 释放内存，然后才它设置为 null. 因为加载 Bitmap 对象的内存空间，一部分是 java 的，一部分 C 的（因为 Bitmap 分配的底层是通过 JNI 调用的 )。 而这个 recyle() 就是针对 C 部分的内存释放。 构造 Adapter 时，没有使用缓存的 convertView ,每次都在创建新的 converView。这里推荐使用 ViewHolder。

