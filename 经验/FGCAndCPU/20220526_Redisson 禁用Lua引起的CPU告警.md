# Redisson 禁用Lua引起的CPU告警

## 背景

前几天部署了一个新工程，当时还挺顺利的。

然而隔天还在上班的路上就收到了运维的信息，CPU告警



## 结论

Redis采用集群模式，且禁用了Lua，而由于我代码里面Redisson操作都是使用异步的，溢出没有打印log，导致有一些线程异常且一直存活者（400-500min）



## 解决方案

redis 集群开启 Lua



## 排除过程

1. CPU过高，第一反应，是不是内存不足，导致频繁FGC，然而这一次错了，内存都是很正常的

2. jstack 去看线程，运维说，有一些Redisson操作的线程占用的CPU很高，刚开始对于这一点我当时是存在疑惑的，因为凌晨请求量很低，没有请求Redisson没有操作，CPU应该会降下来，然而并没有

   ```json
   "redisson-netty-2-8" #56 prio=5 os_prio=0 tid=0x00007fc3280a0800 nid=0x43 runnable [0x00007fc32eefc000]
      java.lang.Thread.State: RUNNABLE
           at java.lang.reflect.Array.get(Native Method)
           at org.redisson.misc.LogHelper.toArrayString(LogHelper.java:121)
           at org.redisson.misc.LogHelper.toString(LogHelper.java:52)
           at org.redisson.misc.LogHelper.toString(LogHelper.java:60)
           at org.redisson.client.handler.CommandDecoder.decode(CommandDecoder.java:353)
           at org.redisson.client.handler.CommandDecoder.decodeCommand(CommandDecoder.java:183)
           at org.redisson.client.handler.CommandDecoder.decode(CommandDecoder.java:122)
           at org.redisson.client.handler.CommandDecoder.decode(CommandDecoder.java:107)
           at io.netty.handler.codec.ByteToMessageDecoder.decodeRemovalReentryProtection(ByteToMessageDecoder.java:508)
           at io.netty.handler.codec.ReplayingDecoder.callDecode(ReplayingDecoder.java:366)
           at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:276)
           at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379)
           at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365)
           at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:357)
           at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1410)
           at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379)
           at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365)
           at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:919)
           at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:166)
   ```

   ![2](D:\project\personal\JavaYang\经验\FGCAndCPU\assets\2.png)

3. 凌晨没有请求，CPU线程还是居高不下，难到有些线程长生不老了？

   top -Hp 去查看，真的有些线程运行了400-500min，且占用的CPU特别高。

4. 怀疑Redisson API 异步操作时，异常导致的，于是将异步操作改成同步操作

5. 改成同步时，马上导出报错（至于由于异步操作，异常时没有打印log, 现在改成同步的，异常马上显现出来）
   ![image-20220525220343586](D:\project\personal\JavaYang\经验\FGCAndCPU\assets\image-20220525220343586.png)

6. 至此得到答案，由于Redis集群禁用Lua引起的，于是开始
   