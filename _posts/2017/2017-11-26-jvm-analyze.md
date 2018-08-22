---
layout: post 
author: 零度
title: JVM菜鸟进阶高手之路十四：分析篇
category: jvm
tags: [JVM]
excerpt: JVM菜鸟进阶高手之路十四：分析篇。
keywords: JVM, GC
---

## 题目回顾
[JVM菜鸟进阶高手之路十三](https://mp.weixin.qq.com/s/5nwYh3ZliPdjHSE9YcC_kQ),问题现象就是相同的代码，jvm参数不一样，表现的现象不一样。
``` java
private static final int _1MB = 1024 * 1024;

	public static void main(String[] args) throws Exception {
		byte[] all1 = new byte[2 * _1MB];
		byte[] all2 = new byte[2 * _1MB];
		byte[] all3 = new byte[2 * _1MB];
		byte[] all4 = new byte[7 * _1MB];
		System.in.read();
	}
```
**jvm参数配置如下：**
``` java
-Xmx20m
-Xms20m
-Xmn10m
-XX:+UseParNewGC 
-XX:+UseConcMarkSweepGC 
-XX:+UseCMSInitiatingOccupancyOnly 
-XX:CMSInitiatingOccupancyFraction=75 
```
通过jstat命令，查看结果如下：

![](http://upload-images.jianshu.io/upload_images/7849276-7d4ec32bd5e7a735.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
关于jstat命令详情可以参考：
[https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)

**jvm参数调整如下：**
``` java
-Xmx20m
-Xms20m
-Xmn10m
-XX:+UseParNewGC 
```
通过jstat命令，查看结果如下：

![](http://upload-images.jianshu.io/upload_images/7849276-c2998bc6cf44417d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 说明
上面的题目仅仅是一个切入点而已，希望通过一个切入点把jvm的一些基础知识刚刚好说明下，顺便解答下上面的现象。


## 内存相关简单说明


![自己画的图，见谅！](http://upload-images.jianshu.io/upload_images/7849276-cfc49dfe0d610dc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**图中参数：**
-Xms设置最小堆空间大小（一般建议和-Xmx一样）。
-Xmx设置最大堆空间大小。
-Xmn设置新生代大小。
-XX:MetaspaceSize设置最小元数据空间大小。
-XX:MaxMetaspaceSize设置最大元数据空间大小。
-Xss设置每个线程的堆栈大小（这里有个故事，3年前用正则表达式，后续有空正则表达式再说）。
>**备注：**tenured空间就用减法操作即可明白，堆空间大小减去年轻代大小就可以了。

**说到这里，下面这个几个参数应该明白了。**
``` java
-Xmx20m
-Xms20m
-Xmn10m
```
>**备注：**参数-XX:SurvivorRatio用来表示s0、s1、eden之间的比例，默认情况下-XX:SurvivorRatio=8表示 s0:s1:eden=1:1:8。

**得出结论：eden=8M,s0=1M,s1=1M,tenured=10M。**




## JVM垃圾回收期组合
还有一个问题需要解决，jvm垃圾回收器方面，下面这个图，我是我的[JVM菜鸟进阶高手之路八（一些细节）](http://mp.weixin.qq.com/s/ZVZFBONZtFrEX4qbz72MlA)，里面的，当时依稀记得这个图应该是飞哥发给我的。
![JVM垃圾回收期组合](http://upload-images.jianshu.io/upload_images/7849276-a2296ebe84fd3f20?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于那个时候jdk9还没有出来，可以去看看我的[JVM菜鸟进阶高手之路十二（jdk9、JVM方面变化， 蹭热度）](http://mp.weixin.qq.com/s/mdaOPsXZFVzhWuGdnFV17A)，虽然有些有些稍微去掉了，但是整体的组合还是影响不大。
![取截取于JVM菜鸟进阶高手之路十二](http://upload-images.jianshu.io/upload_images/7849276-9e5880d52a5c9e22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于上面的2个jvm参数都是基于分代收集算法的（**先不考虑G1**）
- 依据对象的存活周期进行分为新生代，老年代。
- 根据不同代的特点，选取合适的收集算法
  - 新生代，适合复制算法
  - 老年代，适合标记清理或者标记压缩

**复制算法**
- 将原有的内存空间分为两块，每次只使用其中一块，在垃圾回收时，将正在使用的内存中的存活对象复制到未使用的内存块中，之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收。
- 不适用于存活对象较多的场合 如老年代。（年轻代对象基本都是朝生夕灭所以特别适合，由于那样的话复制就少，如果类似老年代有大量存活对象，那么进行复制算法性能就不是特别好了）
>**备注：**使用复制算法的**优点：**每次都是对其中的一块进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况了，使用复制算法的**缺点：**对空间有一定浪费，所以复制空间一般不会特别大。

**标记清除**
标记-清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。在标记阶段，首先先找出根对象，标记所有从根节点开始的可达对象。因此，未被标记的对象就是未被引用的垃圾对象。然后，在清除阶段，清除所有未被标记的对象。
>**备注：**java根对象：
>- 虚拟机栈中引用的对象。
>- 方法区中类静态属性实体引用的对象。
>- 方法区中常量引用的对象。
>- 本地方法栈中JNI引用的对象。
>等等。
标记清除算法**缺点：**标记清除会产生不连续的内存碎片，如果空间内存碎片过多会导致，当程序在运行过程中需要分配空间时找不到足够的连续空间而不得不提前触发一次垃圾收集动作（根据算法不一样效果也不一样）。

**标记压缩**
标记-压缩算法适合用于存活对象较多的场合，如老年代。它在标记-清除算法的基础上做了一些优化。和标记-清除算法一样，标记-压缩算法也首先需要从根节点开始，对所有可达对象做一次标记。但之后，它并不简单的清理未标记的对象，而是将所有的存活对象压缩到内存的一端。之后，清理边界外所有的空间。
>**备注：**这样带来的好处就是不会产生内存碎片问题了。



**上面已经说明了这么多了，我们可以继续说明上题中JVM的其他参数了。**
``` java
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC 
```
-XX:+UseParNewGC 表示新生代使用ParNew并行收集器，-XX:+UseConcMarkSweepGC 表示老年代使用CMS回收器（CMS收集器是基于“标记-清除”算法实现的，**特别提醒由于CMS是标记清除算法实现的所以是存在碎片问题的**）。
![](http://upload-images.jianshu.io/upload_images/7849276-2ba694edfa2162ae?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)可以去看看我的[JVM菜鸟进阶高手之路六（JVM每隔一小时执行一次Full GC）](http://mp.weixin.qq.com/s/ltaUGepOPsMGU8YwyBV8bA)、以及[JVM菜鸟进阶高手之路七（tomcat调优以及tomcat7、8性能对比）](http://mp.weixin.qq.com/s/8qfFejOqY-CbwFUuCGi-IA)图片就取的这两篇里面的。
>**备注：**通过jstat -gcutil pid 查看的FGC这列的时候，CMS gc通常都是**+2**一次的，由于CMS-initial-mark和CMS-remark会stop-the-world。


![](http://upload-images.jianshu.io/upload_images/7849276-7d4ec32bd5e7a735.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以看到这个图的FGC应该没有什么问题了吧。

``` java
-XX:+UseCMSInitiatingOccupancyOnly 
-XX:CMSInitiatingOccupancyFraction=75 
```
还有这2个参数关于cms的，-XX:+UseCMSInitiatingOccupancyOnly表示JVM不基于运行时收集的数据来启动CMS垃圾收集周期通过CMSInitiatingOccupancyFraction的值进行每一次CMS收集，-XX:CMSInitiatingOccupancyFraction=75 表示当老年代的使用率达到阈值75%时会触发CMS GC。
>**备注：**jstat -gcutil可以看出上图的老年代的使用率才60.02%

还有最后一个参数解释:
``` java
-XX:+UseParNewGC 
```
-XX:+UseParNewGC 表示新生代使用ParNew并行收集器，那么老年代呢？
可以让同样参数修改代码执行一次old gc即可看日志有类似**[Tenured:**说明老年代使用的是Serial Old
>**备注：**Serial Old使用的是标记压缩算法。

## 解题
``` java
private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws Exception {
        byte[] all1 = new byte[2 * _1MB];
        byte[] all2 = new byte[2 * _1MB];
        byte[] all3 = new byte[2 * _1MB];
        byte[] all4 = new byte[7 * _1MB];
        System.in.read();
    }
```
**说明：**最后System.in.read();这句可以忽略，只是为了让程序阻塞在那里，不结束，这样好看日志，好看现象而已。

**聪明如你**一下子应该可以看到问题本质:同一份代码，jvm参数堆设置啥的都一样，年轻代gc参数也一样，唯一不同的就在于老年代gc使用上面，而jstat -gcutil图表中FGC没变的应该是正常结果，变了的CMS那个就是意外结果，所以关键点就在CMS上面了。

先来说说all1 、all12、all3、对象实例化开辟空间之后，eden空间都够，他们都在eden空间中，当all4过来的时候，eden空间不够了，需要执行ygc了。
下面有2个问题需要说明，1、如果s0能存的下，可以看看[JVM菜鸟进阶高手之路三：MaxTenuringThreshold](http://mp.weixin.qq.com/s/0O2NaWaMptu_Zt6cy-RWJw)新生代的对象正常情况下**最多**经过多少次YGC的过程会晋升到老生代（CMS情况下默认为6），说到这里可能还需要提一个参数：-XX:TargetSurvivorRatio，可以参考飞哥的：[JVM Survivor行为一探究竟](http://www.jianshu.com/p/f91fde4628a5) 2、如果s0存不下，就是我们这里的情况（由于我们这里s0就是1M而已）所以直接进入到old空间了，**所以可以看出来jstat -gcutil 里面的老年代的比例都是60%几了吧**。
ygc执行完成之后，all4就还可以在eden分配（空间够），**所以可以看出来jstat -gcutil 里面的eden的比例都是89%几了吧**

>**备注：**-XX:PretenureSizeThreshold参数来设置多大的对象直接进入老年代（这个参数其实只对串行回收器和ParNew有效，对ParallelGC无效）。

如果是-Xmx20m -Xms20m -Xmn10m -XX:+UseParNewGC  这套参数，那么结果就是如图可以解释了，并且每个参数比例啥的都可以理解了。
![](http://upload-images.jianshu.io/upload_images/7849276-c2998bc6cf44417d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面来好好解释下这个现象：
![](http://upload-images.jianshu.io/upload_images/7849276-7d4ec32bd5e7a735.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**聪明如你**一下子应该可以看到一个问题，那么就是**时间间隔是每隔2s执行一次**，没错就是2s执行一次。需要说道-XX:CMSWaitDuration(Time in milliseconds that CMS thread waits for young GC)默认值是2s，我们修改为-XX:CMSWaitDuration=5000看看效果：
![CMSWaitDuration变化](http://upload-images.jianshu.io/upload_images/7849276-e35501d27075363e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
看到了吧，修改为5s就是5s执行一次变化了。那么至于为什么会执行呢？？![执行的CMS GC的3中情况](http://upload-images.jianshu.io/upload_images/7849276-418d6cdaa5d23f89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
本题就是当前新生代的对象是否能够全部顺利的晋升到老年代，如果不能，会触发CMS GC。

具体可以看看我比较崇拜的狼哥的分析，一个比我牛逼并且比我努力的大牛。[一个有意思的频繁CMS问题](http://www.jianshu.com/p/a322309b1d90)

------------------

本人其他JVM菜鸟进阶高手之路相关文章或者其他系列文章可以关注公众号【匠心零度】获取更多！！！

**如果读完觉得有收获的话，欢迎点赞、关注、加公众号【匠心零度】。**
