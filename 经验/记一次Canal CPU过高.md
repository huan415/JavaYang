# 记一次Canal CPU过高

## 背景

运维反馈Canal服务CPU太高。
![canal_cpu_top](D:\project\huan415\http\JavaYang\经验\images\canal_cpu_top.jpg)

## 排除结果

由于服务器资源不足，改了Canal内存大小，由于运维不知道参数含义，改错了，堆大小==年轻代大小，导致频繁FGC

![jvm_params](D:\project\huan415\http\JavaYang\经验\images\jvm_params.jpg)



## 排除过程

1. jstack 发现GC线程CPU占用高
   top查进程id
   top -Hp pid 查cpu高的线程
   printf "%x\n" 11062 打印cpu高线程id的16进制
   jstack pid |grep tid -A 30   cpu高线程堆栈信息
   <img src="D:\project\huan415\http\JavaYang\经验\images\canal_cpu_jstack.jpg" alt="canal_cpu_jstack" style="zoom:67%;" />
2. jstat 查看GC情况
   发现Eden和Old都是满的，频繁FGC
   ![canal_cpu__jstat](D:\project\huan415\http\JavaYang\经验\images\canal_cpu_jstat.jpg)
3. Jmap -histo查看哪个类占用内存多（这一步是弯路，对这次排除没什么用，不过也记录一下）
   ![canal_cpu_jmap_histo](D:\project\huan415\http\JavaYang\经验\images\canal_cpu_jmap_histo.jpg)
4. Jmap -heap查看各个代内存情况
   刚看到这里很懵，老年代大小不足0.1M
   ![canal_cpu_jmap_heap](D:\project\huan415\http\JavaYang\经验\images\canal_cpu_jmap_heap.jpg)
5. ps -ef|grep canal 查看启动参数

![canal_cpu_params](D:\project\huan415\http\JavaYang\经验\images\canal_cpu_params.jpg)

整个堆大小=年轻代大小 + 老年代大小 + 永久代大小
如图：堆=1024m，年轻代=1024m，导致老年代没有空间，频繁FGC

至此，调整启动参数，-Xms1024m -Xmx1024m -Xmn512m, cpu就下来了

