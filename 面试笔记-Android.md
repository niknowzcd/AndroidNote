> Activity和Service通信

可以通过bindService的方式,先在Activity里实现一个ServiceConnection接口,并将该接口传递给bindService()方法,在ServiceConnection接口的onServiceConnected()方法 里执行相关操作。

> Service的生命周期与启动方法由什么区别？

- startService()：开启Service，调用者退出后Service仍然存在。
- bindService()：开启Service，调用者退出后Service也随即退出。

> 广播分为哪几种,应用场景是什么?

- 普通广播：调用sendBroadcast()发送，最常用的广播。
- 有序广播：调用sendOrderedBroadcast()，发出去的广播会被广播接受者按照顺序接收，广播接收者按照Priority属性值从大-小排序，Priority属性相同者，动态注册的广播优先，广播接收者还可以 选择对广播进行截断和修改。

> 广播的两种注册方式有什么区别

- 静态注册：常驻系统，不受组件生命周期影响，即便应用退出，广播还是可以被接收，耗电、占内存。
- 动态注册：非常驻，跟随组件的生命变化，组件结束，广播结束。在组件结束前，需要先移除广播，否则容易造成内存泄漏。

> 广播发送和接收的原理

1. 继承BroadcastReceiver，重写onReceive()方法。
1. 通过Binder机制向ActivityManagerService注册广播。
1. 通过Binder机制向ActivityMangerService发送广播。
1. ActivityManagerService查找符合相应条件的广播（IntentFilter/Permission）的BroadcastReceiver，将广播发送到BroadcastReceiver所在的消息队列中。
1. BroadcastReceiver所在消息队列拿到此广播后，回调它的onReceive()方法。

> 广播传输的数据是否有限制，是多少，为什么要限制？

> 遇到过哪些关于Fragment的问题，如何处理的？

- getActivity()空指针：这种情况一般发生在在异步任务里调用getActivity()，而Fragment已经onDetach()，此时就会有空指针，解决方案是在Fragment里使用 一个全局变量mActivity，在onAttach()方法里赋值，这样可能会引起内存泄漏，但是异步任务没有停止的情况下本身就已经可能内存泄漏，相比直接crash，这种方式 显得更妥当一些。
- Fragment视图重叠：在类onCreate()的方法加载Fragment，并且没有判断saveInstanceState==null或if(findFragmentByTag(mFragmentTag) == null)，导致重复加载了同一个Fragment导致重叠。（PS：replace情况下，如果没有加入回退栈，则不判断也不会造成重叠，但建议还是统一判断下）

> Android里的Intent传递的数据有大小限制吗，如何解决？

Intent传递数据大小的限制大概在1M左右，超过这个限制就会静默崩溃。处理方式如下：

- 进程内：EventBus，文件缓存、磁盘缓存。
- 进程间：通过ContentProvider进行款进程数据共享和传递。


