### Android ABI的概念 ###

ABI全称:Application binary interface(应用程序二进制接口),定义了一套规则,允许编译好的二进制目标代码能在所有兼容该ABI的操作系统中无需改动就能运行。

不同的Android手机使用不同的CPU,因此需要提供对应的二进制接口交互规则(即对应的ABI文件)才能进行交互。

部分CPU是能支持多种交互规则，但这是在牺牲性能的前提下所做的兼容。

主流的ABI架构

1. armeabiv-v7a: 第7代及以上的 ARM 处理器。2011年以后的生产的大部分Android设备都使用它。 
2. arm64-v8a: 第8代、64位ARM处理器，很少设备，三星 Galaxy S6是其中之一。 
3. armeabi: 第5代、第6代的ARM处理器，早期的手机用的比较多。 
4. x86: 平板、模拟器用得比较多。 
5. x86_64: 64位的平板。 

### ABI和CPU的关系 ###

当一个应用被安装在设备上时，只有该设备支持的CPU架构对应的.so文件会被安装,如果支持多个ABI架构，会按照优先级进行按照

具体的支持类型如下

**ARMv5(CPU)**:armeabi(ABI)  
**ARMv7**:armeabi,armeabi-v7a  
**ARMv8**:armeabi,armeabi-v7a，arm64-v8a	  
**MIPS**:mips  
**MIPS64**:mips,mips64  
**x86**:x86(1),armeabi(2),armeabi-v7a(3)  
**x86_64**:armeabi,x86,x86_64  

可以看出CPU大都是向前兼容的,但是选择ABI时会有个优先级。

比如X86型的CPU,优先选择x86目录下的.so包，如果存在，就不会再安装其他支持的ABI架构;如果没有x86目录，才会选择armeabi-v7a目录下的.so，最后才会选择armeabi目录下的.so文件。

> ps:X86设备虽然能够运行armeabi下的so库，但可能会损失性能，而且不能保证一定不会发生crash,尤其是有小公司出产的so库

## 使用过程中的问题 ##
### .so文件，放入了优先级低的ABI目录 ###

设备CPU架构是ARMv7，ABI文件是armeabi-v7a,但是放进了armeabi目录中

1.如果项目中有armeabi-v7a目录，目录下没有so库，ARMv7优先加载armeabi-v7a目录，发现没有对应的so库，会报错。

    Caused by: java.lang.UnsatisfiedLinkError

2.项目中只有armeabi的目录,ARMv7会加载armeabi目录,同时加载目录下的so库，相当于加载了一个armeabi规则的so库，因为向前兼容的特性，不会报错，也能运行，但性能会损失。

### .so文件放入优先级高的ABI目录 ###

设备CPU架构是ARMv7，ABI文件是armeabi,但是放进了armeabi-v7a目录中下。

可以被加载，但是能不能被使用？我也不确定。因为armeabi的so库不一定能支持armeabi-v7a定下的接口交互规则。

### 多个第三方的SDK中的ABI文件优先级不一样。 ###

> 两个第三方的SDK中ABI文件优先级不一样，手机加载运行时，会导致优先级低的库，无法被加载

我的手机cpu架构是ARMv7，项目中使用两个第三方SDK：企业A和企业B

企业A：ABI文件是armeabi-v7a，放进armeabi-v7a目录中。   
企业B：ABI文件是armeabi-v5te，放进armeabi目录中。

在运行时，会发现运行后crash，出现如下log信息。

	Caused by: java.lang.UnsatisfiedLinkError
这是因为低优先级的也就是企业B的so包没有被加载进来，也就无法正常使用导致报错。

**解决办法：**

1、使用同一优先级的ABI文件，ABI文件放入优先级相同的ABI目录

企业A：ABI文件是armeabi-v5te，放进armeabi目录中。   
企业B：ABI文件是armeabi-v5te，放进armeabi目录中。   
或 

企业A：ABI文件是armeabi-v7a，放进armeabi-v7a目录中。   
企业B：ABI文件是armeabi-v7a，放进armeabi-v7a目录中。  

2、使用不同优先级的ABI文件，ABI文件放入优先级相同的ABI目录。一般情况不建议这么做。

企业A：ABI文件是armeabi-v7a，但是放进armeabi目录中。   
企业B：ABI文件是armeabi-v5te，放进armeabi目录中。    
或 

企业A：ABI文件是armeabi-v7a，放进armeabi-v7a目录中。   
企业B：ABI文件是armeabi-v5te，但是放进armeabi-v7a目录中。

### so文件的重要法则 ###

处理.so文件时有一条简单却并不知名的重要法则。
你应该尽可能的提供专为每个ABI优化过的.so文件

### NDK的兼容性 ###
使用NDK时，选择app的minsdkVersion对应的编译平台，因为NDK是向后兼容的

### 注意事项 ###
**所有ABI文件夹提供的so要保持一致**

如果我们的应用选择了支持多个ABI，要十分注意：对于每个ABI下的so，但要么全部支持，要么都不支持。不应该混合着使用，而应该为每个ABI目录提供对应的.so文件。

### 参考文章 ###

[安卓app打包的时候还需要兼容armeabi么](https://www.zhihu.com/question/57467260)  
[谈谈Android的so](http://allenfeng.com/2016/11/06/what-you-should-know-about-android-abi-and-so/)  
[ABI和CPU关系的疑难杂症](http://blog.csdn.net/xx326664162/article/details/51167849)  
[Android ABI概念](http://m.blog.csdn.net/stupid56862/article/details/72617065)

### 另外 ###

[个人的github](https://github.com/niknowzcd/AndroidNote)  
[闲暇之余写的故事](https://book.qidian.com/info/1011888583)
