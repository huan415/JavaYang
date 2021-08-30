# binlog、redolog、undolog对比学习

## binlog



## redolog

ib_logfile_*共4个，每个1G。从头开始循环写，当写满之后会进行落盘
write pos 是当前记录的位置，
checkpoint 是当前要擦除的位置
write pos 和checkpoint之间就是空余的空间，当write pos追上checkpoint，这时候不能再更新数据库，redo log file会将数据进行落盘，落盘后checkpoint擦除一些空间之后，再继续

![mysql_log_3](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/mysql_log_3.jpg)

## undolog





log生成时间

下图说明：

1： sync_binlog控制写入binlog时机



2： innodb_flush_log_at_trx_commit控制写入redo log file时机

​       innodb_flush_log_at_trx_commit 的值为0，即每秒才刷到os buffer和磁盘。      
​       innodb_flush_log_at_trx_commit 的值为1，即每次提交都刷日志到磁盘
​       innodb_flush_log_at_trx_commit 的值为2，即每次提交都刷新到os buffer，但每秒才刷入磁盘中。

0、2时:  有点性能比较好。且两者性能差距并不大，因为将log buffer中的日志刷新到os buffer只是内        存数据的转移，并没有太大的开销
              缺点: 可能存在1s内的数据丢失

1时：数据不会丢失，但是性能差点

​      ![](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/mysql_log_2.jpg)

![](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/mysql_log.jpg)



## 对比

|         | 类型     | 层级                         | 功能                              | 读写方式                                                     | 文件                                                         |
| ------- | -------- | ---------------------------- | --------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| binlog  | 物理日志 | service层，所有引擎都有      | 1. 主从库数据同步<br />2.数据备份 | 写：<br />1.顺序写<br />2.事务提交后<br />读：<br />1.从库同步 | mysql-bin.000001                                             |
| redolog | 逻辑日志 | inndoDB存储引擎层,innodb特有 | 持久化---异常宕机数据回复         | 写：<br />1.顺序写<br />2.事务提交完成前<br />读：<br />1.宕机重启数据回复<br />删：循环覆盖写 | ib_logfile0<br />ib_logfile1<br />ib_logfile2<br />ib_logfile3 |
| undolog | 逻辑日志 | inndoDB存储引擎层,innodb特有 | 一致性：MVCC                      | 写：<br />1.随机写<br />2.数据操作前<br />读：<br />1.回滚<br />2.快照读<br />删：purge线程 | ibdata1                                                      |



## crash-safe概念

   保证数据异常宕机，重启之后数据不会丢失，依靠redo log

   两阶段提交：

​           第一阶段： 先写redo log，处于prepare状态

​            第二阶段：写bin log, 写完之后将redo log的状态改为commit状态

1. redolog和binlog之间宕机： 由于还没有commit，所以redolog里的prepare数据没有无效； 
                                                      redolog和binlog数据一致
2. binlog之后，redolog改commit之前宕机： 重启时发现redolog和redolog都有数据，且redolog处于prepare状态，先将redolo改为commit再回复，redolog和binlog数据一致
3. redolog改commit之后：redolog和binlog数据一致