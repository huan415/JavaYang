# Redis持久化

## RDB快照 (snapshot)

dump.rdb文件

配置文件： 
      \# save <seconds> <changes>
      save 900 1          #900s内至少有1次改动
      save 300 10        #300s内至少有10次改动
       save 60 10000    #60s内至少有10000次改动

Redis使用bgsave: redis进程fork子线程来保存快照

|   命令   |    save（同步）    |              bgsave（异步：fork子线程）               |
| :------: | :----------------: | :---------------------------------------------------: |
|   线程   |        同步        |                         异步                          |
| 是否阻塞 |         是         | 否(redis进程fork子线程来处理，<br />fork时稍微有阻塞) |
|   优点   | 不需要消耗额外内存 |                        不阻塞                         |
|   缺点   |        阻塞        |             fork子线程，需要消耗额外内存              |



## AOF（append-only file）

appendonly.aof记录修改的每一条命令，追加在文件末尾
redis重启时，重新执行appendonly.aof里的命令（恢复数据）

### 开启AOF以及fsync频率

AOF步骤
    命令追加:  执行完命令，追加到aof 缓冲区（aof_buf）末尾 --->
    文件写入: 从页缓存（page cache）落盘，刷盘命令: fsync / fdatasync （看如下三种落盘时机）
   文件同步：同步到集群的其他节点

配置文件： 
     appendonly yes   
     \# appendfsync always    \#总是写入aof文件(每次执行命令)，非常安全，但暂用性能不好
    \# appendfsync everysec   #每1s fsync一次，宕机时会损失1s内从数据
    appendfsync no                  #默认，由操作系统自动调度刷磁盘，性能是最好的。缺点：宕机时，数据丢失不可控

### aof文件重写压缩频率

auto-aof-rewrite-percentage 100 \# 上一次重写后，增长达到100%
auto-aof-rewrite-min-size 64mb  \# aof文件大小大于64mb才会重新



## RDB和AOF对比

|   命令   |   RDB    |        AOF         |
| :------: | :------: | :----------------: |
|  优先级  |    低    |         高         |
|   大小   |    小    |    大(每个命令)    |
| 恢复速度 |    快    | 慢(命令一条条执行) |
| 数据丢失 | 容易丢失 |    看fsync频率     |



## 混合持久化

### 背景

redis重启数据恢复，选择哪种持久化文件？
dump.rdb? 数据丢失
appendonly.aof? 恢复速度慢
于是redis4.0推出---混合持久化：结合rdb和aof的优点。

### 开启混合模式

aof-use-rdb-preamble yes \# 默认no 不开启

### 原理

原理：AOF重写那一刻，内存做rdb快照，保存到临时appendonly.aof里，此期间执行的命令也写在临时appendonly.aof(rdb快照后面)，即：appendonly.aof文件前一半是RDB格式、后一半是AOF格式，保存完临时appendonly.aof覆盖原来的文件。
AOF重写期间新执行的命令如何追加到AOF?
答：这期间的命令写在AOF缓冲区和AOF重写缓冲区，并于子线程(写AOF临时文件)间建立通道，将数据告诉子线程----->
       子线程写完RDB之后，子线程从通道中读取数据追加到临时AOF文件，超过20次读取不到数据，则完成AOF重写，覆盖旧AOF文件。然后由主线程负责继续将缓存中的命令追加到新AOF文件。
**重点：重写AOF期间时间太长，页缓存中有太多命令，通过通道分流给子线程写入AOF，否则累计之后由主线程写入AOF，将造成阻塞**
参考：https://www.toutiao.com/i6908602884846666244/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1630643367&app=news_article&utm_source=weixin&utm_medium=toutiao_android&use_new_style=1&req_id=202109031229270101351550513731622C&share_token=978eb382-a661-4f7c-89c7-5e74c866ebfb&group_id=6908602884846666244

