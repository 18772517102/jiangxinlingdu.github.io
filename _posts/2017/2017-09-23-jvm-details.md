---
layout: post 
author: 零度
title: JVM菜鸟进阶高手之路八（一些细节）
category: jvm
tags: [JVM]
excerpt: JVM菜鸟进阶高手之路八（一些细节）。
keywords: JVM, GC
---

## gc日志问题
查看docker环境的gc日志，发现是下面这种情况，很奇怪，一直怀疑是docker环境那里是否有点问题，并没有怀疑配置，之前物理机上面的gc日志都是正常那种。
![](http://upload-images.jianshu.io/upload_images/7849276-0710afc8efbe9da9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>表示很奇怪，后来飞哥告诉我，有没有PrintGCDetails这个参数呀？一看果然，加上之后gc日志就和我们以前看的正常格式一样了。

## Xmn问题
-Xms4g -Xmx4g -Xmn3g 使用cms回收器，查看gc日志，发现一次ygc需要效果大概20s多，平均时间都在10s左右，入下图：![](http://upload-images.jianshu.io/upload_images/7849276-6c85300f50d1ee27?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)查看jmap，并且dump拉下来分析都正常 如图：![](http://upload-images.jianshu.io/upload_images/7849276-447eac0d3d2aebf1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/7849276-4d948d4558bcba6c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)没啥问题啊，觉得特别奇怪，堆还是4G但是xmn去掉了（这里有个误区，以为新老比是1：2（默认） xmn默认就是1.33g 其实不是）查看日志：![](http://upload-images.jianshu.io/upload_images/7849276-8253398f8a277163?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)Young现在760M了, Young代并不是一开始到最大值，也是慢慢扩展，但是这块时间的确没有之前那样的ygc需要10几秒了，再次添加-Xmn1.2g（尴尬了，启动不起来）![](http://upload-images.jianshu.io/upload_images/7849276-1497ea9d7a04c69c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)飞哥告诉我们JVM参数不能有小数，赶紧修改-Xmn1200m正常了。但是ygc的太频繁。![](http://upload-images.jianshu.io/upload_images/7849276-7d299e29af248e9e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)想不要这么频繁，修改-Xmn2g，结果意外了：![](http://upload-images.jianshu.io/upload_images/7849276-25520d3af0bc4f8f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)继续修改回来-Xmn1200m正常了，就是频率高了点。几分钟快700多次了。![](http://upload-images.jianshu.io/upload_images/7849276-65297cff6ac0a7ee?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)可以确定的就是回收速度一定要比分配速度快，从这个来看GC情况还算好。




## 联机事务处理（OLTP）与联机分析处理（OLAP）

**OLTP：On_line Transaction Processing 联机事务处理**
**OLAP：On_line Analytical Processing 联机分析处理**
- OLTP 顾名思义，以业务处理为主。OLAP则是专门为支持复杂的分析操作而设计的，侧重于对决策人员和高层管理人员的决策支持，可以应分析人员的要求快速、灵活地进行大数据量的复杂查询处理，并以一直直观的形式把查询结果提供。
- OLTP与OLAP 的主要区别有以下几点：
1. 所面向的用户和系统：OLTP是面向客户的，由职员或客户进行事务处理或者查询处理。OLAp是向向市场的，由经理、主管和分析人员进行数据分析和决策的。
2. 数据内容：OLTP系统管理当前数据，这些数据通常很琐碎，难以用于决策。OLAP系统管理大量历史数据，提供汇总和聚集机制，并在不同的粒度级别上存储和管理信息，这些特点使得数据适合于决策分析。
3. 数据库设计：通常，OLTP采用ER模型和面向应用的数据库设计，而OLAP系统通常采用星型模式或雪花模式和面向主题的数据库设计。
4. 视图：OLTP系统主要关注一个企业或部门的当前数据，而不涉及历史数据或不同组织的数据。与之相反，OLAP系统常常跨越一个企业的数据库模式的多个版本，OLAP系统也处理来自不同组织的信息，由多个数据源集成的信息。
5. 访问模式：OLTP系统的访问主要由短的原子事务组成，这种系统需要并发控制和恢复机制。而OLAP系统的访问大部份是只读操作，其中大部份是复杂查询。
6. 度量：OLTP专注于日常时实操作，所以以事务吞吐量为度量，OLAP以查询吞吐量和响应时间来度量。针对OLTP系统、OLAP系统 jvm调优还是不一样的。

## OLAP系统垃圾回收器选择
由于系统配置的是cms（和业务系统一样的配置 OLTP系统）这个应该是吞吐量优先，应该不用cms收集器，修改配置，Xmx4g Xms4G Xmn1300m -XX:+UseParallelGC -XX:+UseParallelOldGC ，关于-XX:+UseParallelGC -XX:+UseParallelOldGC是否可以同时写，![这里写图片描述](http://upload-images.jianshu.io/upload_images/7849276-a2296ebe84fd3f20?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
-XX:+UseParallelGC, 新生代ParallelGC回收器，如果老年代不配置，那么老年代串行，只是老年代默认串行而已，再显示老年代UserParallelOldGC, 那老年代就是并行。一起都正常，还没有发现old满gc情况，回头在观察下。
疑惑，感觉xmn这个参数设置很有技巧啊，为什么设置Xmn2g的时候 就会ygc频率的确少了，但是ygc数据长了好几千倍了，为什么设置Xmn1200m或者说Xmn1300m很正常时间都在65ms左右。而且关于YGC真的不好排查，还在实践。希望看见此文章的，如果可以帮助解答疑惑，那么在此表示感谢！！！


-------------------

本人其他JVM菜鸟进阶高手之路相关文章或者其他系列文章可以关注公众号【匠心零度】获取更多！！！

**如果读完觉得有收获的话，欢迎点赞、关注、加公众号【匠心零度】。**