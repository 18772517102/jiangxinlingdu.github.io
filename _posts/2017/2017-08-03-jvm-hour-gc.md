---
layout: post
title: JVM菜鸟进阶高手之路六（JVM每隔一小时执行一次Full GC）
category: jvm
tags: [JVM]
excerpt: JVM菜鸟进阶高手之路六（JVM每隔一小时执行一次Full GC）。
keywords: JVM, GC
---

上次分析详细地址在：[http://www.jiangxinlingdu.com/jvm/2017/07/28/jvm-rmi.html](http://www.jiangxinlingdu.com/jvm/2017/07/28/jvm-rmi.html)
以为上次问题是rmi的问题就此结束了，但是问题并没有结束，其实本次问题不是rmi问题导致的，但是rmi也的确可能会有sys.gc fullgc问题。
查看GC统计汇总情况：
``` java 
jstat -gcutil pid  3s 30 
```
![](http://upload-images.jianshu.io/upload_images/7849276-13093ee1679f7474?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)参考gc，发现大概一小时运行一次FGC，特别奇怪，笨神一看这样的问题就知道是**system gc**导致的![](http://upload-images.jianshu.io/upload_images/7849276-d56daf6d42520413?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![](http://upload-images.jianshu.io/upload_images/7849276-9ccd5ce2c50b59a8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)System.gc()的默认效果是引发一次stop-the-world的full GC，对整个GC堆做收集。用**-XX:+DisableExplicitGC**参数后，System.gc()的调用就会变成一个空调用，完全不会触发任何GC。

问题来了 如果直接使用-XX:+DisableExplicitGC就没有下面的任何事情了，在测试过程中的确使用了该参数，的确不会有full gc了。但是有写堆外空间的释放是需要涉及到System.gc的，如果禁用可能见到direct memory的OOM类似的异常，由于各种原因我们的环境没有禁用。由于没有禁用，添加参数**-XX:+ExplicitGCInvokesConcurrent** 该方法可以指定System.gc()采用 CMS 算法，FGC时停机时间会变短，但是CMS GC次数不会变。通过该参数 经过观察日志效果比Full GC要好些。

看到这里一切都挺完美，后面开始要到高潮了，纠结…………看上篇文章里面有说一直以为是rmi问题，就查找资料想让时间延迟下执行，
``` java 
-Dsun.rmi.dgc.client.gcInterval=36000000 
-Dsun.rmi.dgc.server.gcInterval=36000000
```
单位是毫秒，可适当延长触发FGC的定时时间间隔。 文中配置为10小时。通过jstack查看JVM线程
``` java 
   GC Daemon" daemon prio=10 tid=0x00007f91bcccf800 nid=0x37f0 in Object.wait() [0x00007f9182706000]  
   java.lang.Thread.State: TIMED_WAITING (on object monitor)  
    at java.lang.Object.wait(Native Method)  
    - waiting on <0x0000000600021a48> (a sun.misc.GC$LatencyLock)  
    at sun.misc.GC$Daemon.run(GC.java:117)  
    - locked <0x0000000600021a48> (a sun.misc.GC$LatencyLock)  
  
   Locked ownable synchronizers:  
    - None  
```

![](http://upload-images.jianshu.io/upload_images/7849276-9979cbb91bf263bd?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)笨神告诉我们，如果该线程一旦创建了就会间隔的调用gc了，所以就会存在定时继续full gc问题。一直通过观察居然没有效果，还是会一小时执行一个full gc。通过gc日志可以出出来：![](http://upload-images.jianshu.io/upload_images/7849276-311d7cc5e03edfeb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**而且old区6g才占2.5G就background cms gc了**![](http://upload-images.jianshu.io/upload_images/7849276-2dcff39db65661f5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)修改为cms的时候，每次System.gc 一次full gc的时候cms gc还会加2的，触发的是background cms gc如果不是后台就会一次，cms过程如下：![](http://upload-images.jianshu.io/upload_images/7849276-2ba694edfa2162ae?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)里面有一些概念比较重要，并行和并发的定义。gc这个场景下 你可以这么记 并发表示gc线程可以和业务线程同时跑 并行表示gc线程跑的时候业务线程都全不能跑 。

在Java 9 中将 Java 8 默认的 GC 从 Parallel GC 改为 G1，cms不是和ps比速度的，cms是低延时垃圾回收器。

开始纠结了笨神告诉需要通过btrace跟踪下就很容易定位到问题那里调用了System.gc,根据ak大神提供的地址，之后阿飞给了我关于btrace新的github地址：https://github.com/btraceio/btrace。 以及一些Sample参考：https://github.com/btraceio/btrace/tree/master/samples。 github官网很多参考样例，基本上能覆盖常用的需求编写了查看gc代码如下：
``` java
@OnMethod(clazz = "java.lang.System", method = "gc")    
    public static void onSystemGC() {    
        println("entered System.gc()");    
        jstack();    
} 
```

![](http://upload-images.jianshu.io/upload_images/7849276-89c47111971a0841?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)在本地调用模拟结果如下：![](http://upload-images.jianshu.io/upload_images/7849276-b1bbafeb22d47f93?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)放到环境进行观察，也获取到了结果如下：![](http://upload-images.jianshu.io/upload_images/7849276-2005485c1db94218?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)打印到这里 知道是sun.misc.GC调用的，在这里走了很多弯路了，后来我把rmi去掉了，但是还是一小时一次通过日志观察，后来搜索发现tomcat文章，![](http://upload-images.jianshu.io/upload_images/7849276-5212f088bc751c62?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![](http://upload-images.jianshu.io/upload_images/7849276-d8cb2cb4222783e0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![](http://upload-images.jianshu.io/upload_images/7849276-da08bf0502541834?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![](http://upload-images.jianshu.io/upload_images/7849276-2a20baf8c27f6870?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)的确有，开始也没有注意，以为是这个原因，修改了重试还是不行，后是一波折过程，后来查看jar文件，的确不是一小时了，
![](http://upload-images.jianshu.io/upload_images/7849276-9f864a77a478c137?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
后来又看见![](http://upload-images.jianshu.io/upload_images/7849276-a0502f3f1b5f0867?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)以为这个包问题，又是一波修改，发现还是一小时执行一次通过日志观察，此时我已经无语了，不过还好在我的坚持下，还是把问题找到了，由于我把项目去掉跑不会有，那么感觉和项目有关，但是代码里面的确没有调用，我怀疑是否是其他jar里面的问题呢？我把所有的jar都查了一遍，的确发现问题了。![](http://upload-images.jianshu.io/upload_images/7849276-2e8d1ee8d82ee76e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)查看该jar![](http://upload-images.jianshu.io/upload_images/7849276-e0a07b20771d8bcc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)由于包的确有点老了，里面的确是这样，和上面的tomcat那个bug很像，我下载了一个新版本查看，发现的确优化了。![](http://upload-images.jianshu.io/upload_images/7849276-f4b4820d93363d1b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)新版本里面变成了10个小时一次了，而且可以通过jvm参数让其进行关闭，
**-Dorg.apache.cxf.JDKBugHacks.gcRequestLatency=true**即可。这次的这个一小时问题排除就结束了，还需要修改代码，后续继续观察，在此过程中，ak大神和阿飞都告诉我关于ygc时间问题，的确这个还一直在实验，希望优化的更好，内容很多一直也在学习，定位问题就是需要大胆的猜之后试之后优化修改记录。后续会分享关于ygc时间长问题，推荐一款在线分析gc的好工具析，http://gceasy.io。  非常棒，在此再次感谢笨神，阿飞哥，ak大神的指导。

------------------

本人其他JVM菜鸟进阶高手之路相关文章或者其他系列文章可以关注公众号【匠心零度】获取更多！！！

**如果读完觉得有收获的话，欢迎点赞、关注、加公众号【匠心零度】。**