# MVCC 多版本并发控制----可见性算法

1. 当前读： 读取的是最新版本（insert; update; delete; select ...lock in share mode; select ... for update）

2. 快照读： 读取的是历史版本(select )

   

读已提交（RC）与可重复读（RR）比较

|                | 读到的记录           | readview                                                     |
| -------------- | -------------------- | ------------------------------------------------------------ |
| 读已提交（RC） | 读到的是最新的记录   | 每次快照读都生成新的readview                                 |
| 可重复读（RR） | 读到的是MVCC历史记录 | 只有在第一次快照读时才生成readview，注意（insert、update,delete等也会涉及到快照读） |



## MVCC原理

1. 隐藏字段

   |        DB_TRX_ID         | DB_ROW_ID |        DB_ROLL_PTR        |
   | :----------------------: | :-------: | :-----------------------: |
   | 创建或者最后一次修改的id | 隐藏主键  | 回滚指针，指向undolog地址 |

   

2. undolog 历史数据----线性链表

   形成链表：链头--最新历史数据    链尾：最早历史记录

   假设：unodlog的地址：ox125-->ox124-->ox123>null

   | name    | DB_TRX_ID | DB_ROW_ID | DB_ROLL_PTR |
   | ------- | :-------: | :-------: | :---------: |
   | yangyc4 |     4     |     1     |    ox125    |

| name    | DB_TRX_ID | DB_ROW_ID | DB_ROLL_PTR |
| ------- | :-------: | :-------: | :---------: |
| yangyc3 |     3     |     1     |    ox124    |

| name    | DB_TRX_ID | DB_ROW_ID | DB_ROLL_PTR |
| ------- | :-------: | :-------: | :---------: |
| yangyc2 |     2     |     1     |    ox123    |

| name    | DB_TRX_ID | DB_ROW_ID | DB_ROLL_PTR |
| ------- | :-------: | :-------: | :---------: |
| yangyc1 |     1     |     1     |    null     |

   

3. readview 快照读产生的读视图

   |   trx_list   |    up_limit_id     |     low_limit_id     |
   | :----------: | :----------------: | :------------------: |
   | 活跃的事务id | 列表中事务最小的id | 未分配的下一个事务id |

   

​    原则：

1. 首先，DB_TRX_ID < up_limit_id ，可以看到DB_TRX_ID所在的记录
                                                              否则，下一个比较’
2. 其次，DB_TRX_ID >= low_limit_id, DB_TRX_ID在Read view生长之后，所以当前事务不可见
                                                              否则，下一个比较’
3. 最后，DB_TRX_ID 在trx_list中，则Read view时，当前事务还在活跃状态，没有commit，所以当前事务看不到。否则能看到

