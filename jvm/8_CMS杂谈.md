# CMS杂谈

## **concurrent mode failure（并发失败）**

1. CMS 收集器无法回收浮动垃圾（并发标记和并发清理阶段产生的垃圾）
2. CMS 在并发阶段（并发标记和并发清理）会一直产生垃圾，所以它不能像其他垃圾回收器一样等到老年代几乎已经满了再进行FGC，必须预留一定的空间供并发阶段使用
3. 当预留的空间太小，即并发阶段产生的垃圾放不下，则会发生并发失败（concurrent mode failure）。此时虚拟机采用改用其他垃圾回收器----Serial Old（STW时间长）
4. 老年代空间达到多少会触发 FGC ? -XX： CMSInitiatingOccu-pancyFraction指定百分比
   JDK5：默认68%
   JDK6：默认92%
5. -XX： CMSInitiatingOccu-pancyFraction指定触发FGC的百分比
   百分比太高：容易发生concurrent mode failure，Serial Old导致STW时间长
   百分比太低:  频繁发生FGC

**解决办法：**

1. -XX:+UseCMSInitiatingOccupancyOnly (见下面详解)
2. -XX:CMSInitiatingOccupancyFraction=60 

##  **promotion failed**

Minor GC 时，由于 Survivor 空间不足，对象要进入老年代，而此时老年代的空间也不足。
一般都是因为碎片比较多，新生代 --> 老年代的对象比较大，找不到连续的区域放这个对象导致的。
**解决办法：**  设置 XX:CMSFullGCsBeforeCompaction (见下面详解)





##  -XX:CMSInitiatingOccupancyFraction=70 和-XX:+UseCMSInitiatingOccupancyOnly

-XX:+UseCMSInitiatingOccupancyOnly  如果有指定，严格按照XX:CMSInitiatingOccupancyFraction设定的百分比进行GC

​                                                                      如果不指定，第一次按照XX:CMSInitiatingOccupancyFraction设定的百分比，后续的比例虚拟机会自动调整

 -XX:CMSInitiatingOccupancyFraction=70   指定触发FGC的百分比， 即：老年代空间达到多少百分比就开始FGC



## 

## -XX:+UseCMS-CompactAtFullCollection 和 -XX:CMSFullGCsBeforeCompaction

-XX:+UseCMS-CompactAtFullCollection 开启碎片整理, 与下面的参数配合使用

-XX:CMSFullGCsBeforeCompaction:      CMS 执行若干次 FGC 只会，再下一次 FGC 执行进行碎片整理（默认0，即：每次FGC之前都会进行碎片整理）
注意：

1. XX:CMSFullGCsBeforeCompaction=10 是指每间隔10次做一次碎片整理，而不是每10次

参考：
           1. [【拥抱大厂系列】面试官100%会严刑拷打的 CMS 垃圾回收器，下次面试就拿这篇文章怼回去](https://zhuanlan.zhihu.com/p/137056981)
              2. [22-大厂面试题：Con-current Mode Failure如何导致以及解决](https://zhuanlan.zhihu.com/p/400240403)          
              3. [-XX:CMSInitiatingOccupancyFraction=70 和-XX:+UseCMSInitiatingOccupancyOnly](https://blog.csdn.net/varyall/article/details/80041068)

​            

