---
layout: post
title: JVM菜鸟进阶高手之路十三（等你来战！！！）
category: jvm
tags: [JVM]
excerpt: JVM菜鸟进阶高手之路十三（等你来战！！！）。
keywords: JVM, GC
---

**前几天有个朋友问了我个问题，下面给大家分享下，希望大家积极在评论区进行评论留言，等你来战！！！**

先来个趣味题，热身下，引出后面的jvm题目。
## 地上的影子是那个人的？
![图片来自网络](http://upload-images.jianshu.io/upload_images/7849276-92d3f09fae12304e.JPEG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**地上的影子是左边人的还是右边人的？**

哈哈哈，知道你一定挺纠结的吧。下面看看今天的jvm这个问题呢？
## 这个JVM现象该如何解释呢？
**代码如下：**
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

**为什么呢?**，希望大家积极在评论区进行评论留言，等你来战！！！



--------------------------
## 本人其他JVM菜鸟进阶高手之路相关文章
- [JVM菜鸟进阶高手之路（jdk9、JVM方面变化， 蹭热度）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483711&idx=1&sn=2e2e702f68d84d3c9dc9173b56de603c&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路十一（eden survivor分配问题）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483702&idx=1&sn=310d34a5a10f12959fd58581507e7b3f&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路十（基础知识开场白）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483703&idx=1&sn=d24bf3f2fe869e272e0c1543cc8ebf42&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路九（解惑）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483673&idx=1&sn=bc11600868fd7d659bb6792101c1fd20&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路八（一些细节）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483688&idx=1&sn=4d9023d7f556b308070915df999ce4a9&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路七（tomcat调优以及tomcat7、8性能对比）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483695&idx=1&sn=3afc6e930ed602dad45bc1ae75afef7e&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路六（JVM每隔一小时执行一次Full GC）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483691&idx=1&sn=61e77c56a654985eb44c3e8f91f088a9&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路五](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483691&idx=2&sn=e80b391d9bec94e24cad615dab156d46&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路四](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483680&idx=4&sn=d590b1dcb0533027b0e632fe6c1f4d21&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路三](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483680&idx=3&sn=0f0d3ca480c28470ef41686ad7d61a6c&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路二](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483680&idx=2&sn=c83333705f423ad344c61a8541457897&scene=19#wechat_redirect)
- [JVM菜鸟进阶高手之路一(一次与笨神，阿飞近距离接触修改JVM）](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247483680&idx=1&sn=079d801a792b03016704dfcc99158b6e&scene=19#wechat_redirect)

----------------------------------------------------------------
![图片来自网络](http://upload-images.jianshu.io/upload_images/7849276-bf57531b1e784827.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

-------------------

本人其他JVM菜鸟进阶高手之路相关文章或者其他系列文章可以关注公众号【匠心零度】获取更多！！！

**如果读完觉得有收获的话，欢迎点赞、关注、加公众号【匠心零度】。**