# Redis常见零碎知识点

## Redis为什么快？

1. 基于内存操作，时间复杂读O(1)
2. 多路复用，NIO
3. 单线程，避免上下文切换开销
4. 数据结构简单

## Redis删除策略

1. redis如何清除过期数据（Redis是使用**定期删除** + **惰性删除** 两者配合的过期策略。）

   * 定时删除
     **概念**：为每个键设置一个定时器，当key设置有过期时间，且过期时间到达时，由定时器任务立即执行对键的删除操作
     **优点：** 到时就删，节约内存
     **缺点：** CPU压力大，影响Redis的响应时间和吞吐量

   * 定期删除
     **概念**：redis默认是每隔100ms就**随机抽取**一些设置了过期时间的key，检查其是否过期，如果过期就删除
     **优点：** 节约CPU
     **缺点：** 占用内存（过期的数据仍占用内存）
     为什么要随机抽取：每次都全库扫描，非常浪费CPU
     
   * 惰性删除
     **概念**：等下次访问数据才删除（过期时间到了也不处理）
     **优点：** CPU性能占用设置有峰值，检测频度可以自定义设置

     ​           内存压力不是很大，长期占用内存的冷数据会被持续清理
     **缺点：** 占用内存（过期的数据仍占用内存）

2. Redis逐出算法
   Redis每执行一条命令，调用freeMemoryIfNeeded()检测内存是否充足，内存不足则清理空间。清理不成功,则循环清理。
   删除策略：

   * 检测易失数据
     volatile-lru (Least Recently Used)：最长时间未被使用
     volatile-lfu (Least Frequently Used)：最近使用最少
     volatile-ttl：将要过期
     volatile-random：任意
   * 检测全库数据
     allkeys-lru：最长时间未被使用
     allkeys-lfu：最近使用最少
     allkeys-random：任意
   * 放弃数据驱逐
     no-enviction：禁止驱逐数据，会引发错误OOM
   
   ```
   noeviction:返回错误当内存限制达到并且客户端尝试执行会让更多内存被使用的命令（大部分的写入指令，但DEL和几个例外）
   
   allkeys-lru: 尝试回收最少使用的键（LRU），使得新添加的数据有空间存放。
   
   volatile-lru: 尝试回收最少使用的键（LRU），但仅限于在过期集合的键,使得新添加的数据有空间存放。
   
   allkeys-random: 回收随机的键使得新添加的数据有空间存放。
   
   volatile-random: 回收随机的键使得新添加的数据有空间存放，但仅限于在过期集合的键。
   
   volatile-ttl: 回收在过期集合的键，并且优先回收存活时间（TTL）较短的键,使得新添加的数据有空间存放。
   ```
   
   

## 跳跃表

链表：要查找一个元素，从头开始遍历链表，效率低（时间复杂读：O(n) ）
跳跃表优化：在链表上创建索引，每拉个节点提取一个节点到上层，把提取出来的新元素(只存储关键字和指针)组成新链表作为索引。（空间换时间）
参考url: https://www.cnblogs.com/hunternet/p/11248192.html



## geospatial 地理位置 （底层实现原理Zset）

1. 场景：朋友的定位、附近的人、打车距离、推算地理位置信息、两地之间距离、方圆几里的人

2. 命令：

   ```java
   // 添加地理位置(经纬度有范围限制，超出会报错)
   GEOADD china:city 经度 维度 厦门
   // 获取指定的经纬度
   GEOPOS china:city 厦门
   // 两个给定位置之间的距离
   GEODIST  china:city 厦门 漳州
   // 给定的经纬度为中心，找出某一半径内的元素
   GEOradius china:city 经度 维度 10 km withcood
   // 找出位于指定元素周围的元素
   GEOradiusByMenber 经度 维度 厦门 10 km
   // 返回一个或多个位置的GeoHash表示
   GEOHash
   ```

   



## 位图bitmap（适合两个状态）

1. 位图不是特殊的数据结构，而是普通的字符串（byte 数组）
2. 数据量大时节省空间
3. 举例：
   1. 打卡（1:打开，2:没打卡），一年365位的空间
   2. 布隆过滤器

```shell
setbit yangyc:huan415:uerid 0 1
getbit yangyc:huan415:uerid 0
bitcount yangyc:huan415:uerid
```





## HyperLogLog 基数统计算法

1. 统计 PV，最终就可以统计出所有的 PV 数据
   (如果用set集的sadd，那么统计时数据量太大，且浪费空间)
2. 举例：
   1. 网页UV, 一个人范围网页多少次（同一个人要去重）
      来一个用户 ID，就将用户 ID 塞进去就是（pfadd）
3. 有点
   1. 占用内存空间大小固定，12kb

```shell
# 1.pfadd   增加计数（类似于set集合的sadd）
# 1.pfcount 获取计数（类似于set集合的scard）
```

## SDS（simple dynamic string 动态字符串）

1. 于C语言字符串的区别

## keys与Scan

1. scan概念
   scan命令是一个基于游标的迭代器，每次被调用之后， 都会向用户返回一个新的游标， 用户在下次迭代时需要使用这个新游标作为 SCAN 命令的游标参数， 以此来延续之前的迭代过程
2. scan优点
   * 分批次进行，不会阻塞线程
   * limit参数，可以控制每次返回结果的最大条数
3. scan缺点
   * 结果有可能重复
   * 这个模糊匹配过程，scan总时长 > keys总时长（每次scan时间 < keys时间）
4. scan比Scan最大优点：不会阻塞。但是总时长比较长

## 渐进式rehash

1. Redis有两个Hash表，一个旧表、一个新表。Rehash过程，即：将旧表数据逐步迁移到新表中（以bucket为单位）
2. 触发rehash的条件
   负载因子 = 哈希表已保存节点数量 / 哈希表大小
   url: https://zhuanlan.zhihu.com/p/358366217
   * 触发扩容操作
     * 没有在BGSAVE 命令或者 BGREWRITEAOF 命令，负载因子 >= 1
     * 正在BGSAVE 命令或者 BGREWRITEAOF 命令，负载因子 >= 5
   * 触发收缩操作
     * 负载因子小于0.1

## Redis实现延迟队列

* sortedset结构：与Sets类型极为相似，不允许重复。
                               每一个成员都有score，按score排序（score可以重复）
* zadd命令：加入成员(包括score)到有序集合
                       zadd key score value
* zrangebyscore命令：返回有序集合中指定分数区间的成员列表（按score排序）
                       zrangebyscore key min max
* 延迟队列：
  1. zadd key  执行时间作为score
  2. zrangebyscore 获取N秒内的数据轮询进行处理

![](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/3_redis_zadd_1.png)

![](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/3_redis_zadd_2.png)
