> EventBus这是一个目前被广泛使用的,用于在不同的界面(Activity/Fragment)之间传递数据的三方库,其简单的使用深受广大开发者喜欢。

相比起Bundle或者接口回调,EventBus使用起来更加简单快捷,但有一点需要注意,再使用EventBus的时候,你需要对自己业务中的通知分发有很清晰的了解，不然很容易导致分发过多的无用通知,导致性能的消耗.

**本文会对EventBus做一个简单的介绍**

### 简单的使用 ###
**1.oncreate()中注册**

    EventBus.getDefault().register(this);

**2.onDestroy()销毁**

    EventBus.getDefault().unregister(this);

**3.使用`@Subscribe`注解方法,用来接收事件通知**

	@Subscribe
    public void onMainEvent(String str){
        System.out.println("event = "+str);
    }

**4.发送通知**

    EventBus.getDefault().post("12456");

上述4个步骤就能完成一次简单的事件分发,这也是EventBus被广泛使用的部分原因。

### 源码解读 ###


> 在深入源码之前先解释几个主要的成员  

**Subscription.class (订阅信息)**  

	final class Subscription {
		//订阅者对象,一般情况下多为activity/fragment等
	    final Object subscriber;
		//订阅者方法对象,主要包括订阅的
	    final SubscriberMethod subscriberMethod;
	}

**SubscriberMethod.class (订阅者方法对象)**

	public class SubscriberMethod {
		//方法
	    final Method method;
		//线程模式
	    final ThreadMode threadMode;
		//事件类型
	    final Class<?> eventType;
		//优先级
	    final int priority;
		//是否是粘性事件
	    final boolean sticky;
	}

**FindState.class (查找状态,主要用来保存查找过程中的变量和结果)**
	
	 static class FindState {
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
		//事件type为key,method为value
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
		//method为key,订阅者class对象为value
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        final StringBuilder methodKeyBuilder = new StringBuilder(128);
	}


另外还有几个比较重要的Map对象  

1. **subscriptionsByEventType**  type为key,`CopyOnWriteArrayList<Subscription>`为value,从这里可以看出EventBus中一个type类型可以对应很多个订阅者。
2. **typesBySubscriber** 则刚好相反,Subscription为key,Subscription内的所有事件type为value
3. **METHOD_CACHE** class类对象为key,List<SubscriberMethod>为value,注意这里的是SubscriberMethod,而上述的是 Subscription，
其中值得注意的是`METHOD_CACHE`采用的是`ConcurrentHashMap()`这个数据模型,对于多并发做了一定的优化。



**接下来我们以前看看这个强大的函数库的内部实现原理。**
	
### register() ###

	public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

**register过程的主要工作**  
1.获取订阅者的类名 (比如MainActivity)  
2.通过`findSubscriberMethods`方法查找订阅者的订阅方法 
(@Subscribe注解的并且是Public修饰的)  
2.1 查找订阅方法

	List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
		//ignoreGeneratedIndex属性表示是否忽略注解器生成的MyEventBusIndex,默认情况下为false
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
		//如果订阅者类中没有被 @Subscribe且public声明的方法就会报异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }

### findUsingInfo() ###

	 List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

其中以线程池的形式获取FindState对象,并初始化Subscriber订阅者对象

	FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
	
正常情况下第一次使用`if (findState.subscriberInfo != null)`这个判断会为false,接下来进入`findUsingReflectionInSingleClass(findState);`流程

另外需要注意的是每一次的循环都会调用`findState.moveToSuperclass()`检索父类的方法,所以对于一个订阅者来说,子类和父类中的方法都会被检索到,顺序是子类->父类

**findUsingReflectionInSingleClass()**

	 private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }

1. 通过放射获取所有的类中所有的方法数
2. 过滤方法 1.public修饰 2.非静态，非抽象 3.一个参数 4.函数拥有@Subscribe的注解
3. `findState.checkAdd(method, eventType)`校验是否可以加入到list中

`findState.checkAdd(method, eventType)`这里分两种情况  

1. `anyMethodByEventType()`中没有的直接返回true  
2. `anyMethodByEventType()`中有的,做2次校验,
 
这次根据 **方法名>参数名**进行完整校验,因为同一个类中同名同参的函数是不存在的,而同名不同参的在前一步已经被过滤了，所以这里确保在一个类中不会重复注册.  

但如果子类重写父类的方法的话,就会出现相同的methodKey。这时EventBus会做一次验证,并保留子类的订阅信息。由于扫描是由子类向父类的顺序，故此时应当保留methodClassOld而忽略methodClass。如果代码上的注释 `Revert the put`

    if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
        // Only add if not already found in a sub class
        return true;
    } else {
        // Revert the put, old class is further down the class hierarchy
        subscriberClassByMethodKey.put(methodKey, methodClassOld);
        return false;
    }

### checkAdd() 之后 ###

    ThreadMode threadMode = subscribeAnnotation.threadMode();
    findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));

添加到findState中,以一个arrayList保存

接下来回到 `getMethodsAndRelease()`方法中,return 一个方法集合
List<SubscriberMethod>

	private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }


3.回到开头的 `register()`方法中

	for (SubscriberMethod subscriberMethod : subscriberMethods) {
        subscribe(subscriber, subscriberMethod);
    }

相对而言注册流程还是比较简单的,主要是让最开始提到的两个map(subscriptionsByEventType和typesBySubscriber)里面填加数据,留着事件分发时候使用。最多加一些优先级/粘性事件的判断。

	 private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }	


### post流程 ###

同样先介绍几个重要的类

**PostingThreadState.class  (事件分发状态)**

 	final static class PostingThreadState {
		//事件队列
        final List<Object> eventQueue = new ArrayList<>();
        //是否正在分发,防止并发状态下同一个事件发出多次
		boolean isPosting;
		//是否在主线程
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
	
**ThreadLocal.class (线程数据的存储类)**  
在指定线程存储数据,数据存储之后，只有在指定线程才能获取到之前存储过的数据。

	
**post流程**

	public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }

核心部分只有下面这句,遍历发送eventQueue队列中的event,并且移除已经发送的event
	
	postSingleEvent(eventQueue.remove(0), postingState);

**postSingleEvent.class**

	private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }

`eventInheritance`变量默认情况下被赋值为true,其中`lookupAllEventTypes`函数会遍历eventclass,得到其父类和接口的class类,事件派发的核心部分在`postSingleEventForEventType()`

**postSingleEventForEventType.class**

	 private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }

这个函数的逻辑也是十分简单,根据eventClass从`subscriptionsByEventType`列表中获取订阅者列表,接着遍历订阅者列表,以此将event回调。读了这里时,发现将eventClass作为key,而不是event作为key,估计是因为class对象能追溯到其父类和接口实现吧。

到此,post流程也结束了,比起注册流程还要简单。

### 线程调度 postToSubscription() ###


> 几种poster类型

**1.mainThreadPoster** 创建于**HandlerPoster**

**HandlerPoster.class**
	
	主要变量
	//队列
	private final PendingPostQueue queue;
	//最大存在秒数 通常为10s,超过则会报错,这就跟广播的onReciver回调10s没处理完成就会报ANR错误有些类似
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
	//标志是否活动可用的
    private boolean handlerActive;
	
核心逻辑 `handleMessage`

	@Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
遍历队列,执行`eventBus.invokeSubscriber(pendingPost)`方法
	
  	void invokeSubscriber(Subscription subscription, Object event) {
        try {
			//通过反射的方式，直接调用订阅该事件方法。
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }

**2.BackgroundPoster:**实现 Runnable,会将PendingPostQueue队列中所有的事件分发出去
  
**3.AsyncPoster:**同样实现 Runnable,只会分发一个PendingPostQueue队列中的事件


**postToSubscription.class**

	private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
**逻辑也很好理解,相同线程时直接`invokeSubscriber`反射回调,不同线程则发到相同线程去回调。**

### unregister流程 ###

    public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }

主要工作分两步 

1. 移除订阅者的所有订阅信息   
2. 移除订阅者和其所有Event的关联

> 另外看过的一篇博文上写了有关于EventBus的优化操作,记录下来

1. eventInheritance默认为true,会遍历class对应的父类和接口对象,如果程序中没有使用到事件的继承关系,可以将该值设为false
2. ignoreGeneratedIndex表示是否使用生成的APT代码去优化查找事件的过程,如果项目中没有接入EventBus的APT，也可以将其设置为false


### 补充 ###

> hashMap.put()

hashMap.put() 会根据key返回对应的value值  
如果put的时候没有对应的key值,则添加到map中,如果有 先返回后添加

如

	map.put("222", "222");
    String value2=map.put("222","333");

此时的value2为"222"，而map中 key="222"的value为"333"	

> instanceof, isinstance,isAssignableFrom的区别

instanceof运算符 只被用于对象引用变量，检查左边的被测试对象 是不是 右边类或接口的 实例化。如果被测对象是null值，则测试结果总是false。 
形象地：自身实例或子类实例 instanceof 自身类  返回true 

      String s=new String("javaisland"); 
      System.out.println(s instanceof String); //true 

Class类的isInstance(Object obj)方法，obj是被测试的对象，如果obj是调用这个方法的class或接口 的实例，则返回true。这个方法是instanceof运算符的动态等价。 
形象地：自身类.class.isInstance(自身实例或子类实例)  返回true 

      String s=new String("javaisland"); 
      System.out.println(String.class.isInstance(s)); //true 

Class类的isAssignableFrom(Class cls)方法，如果调用这个方法的class或接口 与 参数cls表示的类或接口相同，或者是参数cls表示的类或接口的父类，则返回true。 
形象地：自身类.class.isAssignableFrom(自身类或子类.class)  返回true 
例：

	System.out.println(ArrayList.class.isAssignableFrom(Object.class));  //false 
    System.out.println(Object.class.isAssignableFrom(ArrayList.class));  //true

### 参考文章 ###

[EventBus源码分析](https://juejin.im/post/5ad3e365f265da239c7bcf2b)

[EventBus](https://juejin.im/post/5ab7775bf265da23906bfadc)