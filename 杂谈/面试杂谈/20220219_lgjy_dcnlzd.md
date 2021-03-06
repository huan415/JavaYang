# lgjy_dcnlzd

背景：2022-02-19 参加大厂能力诊断卡记录部分经验

## MQ

### RabbitMQ

1. RabbitMQ 怎么持久化
   rabbitmq 整个消息投递的路径为：producer--->rabbitmq broker--->exchange--->queue--->consumer

   * 消息从 producer 到 exchange 则会返回一个 confirmCallback 。
   * 消息从 exchange-->queue 投递失败则会返回一个 returnCallback 。
     我们将利用这两个 callback 控制消息的可靠性投递
   * queue-->consume   ack 和 nack 机制

   准备工作：

   * 生产者：连接 RabbitMQ、获取信道、声明交换机、创建消息、发布消息、关闭信道、关闭连接
   * 消费者：连接 RabbitMQ、获取信道、声明交换机、声明队列、把队列和交换机绑定起来、消费消息、关闭信道、关闭连接
   * 生产者发送消息==》RabbitMQ
     那么如何保证消息有持久化到磁盘，而不是持久化之前就已经宕机。
     * 保证消息到达RabbitMQ方式
       * 事务确认机制（发送一条消息，将其持久化到交换机时，将消息提交到持久化日志后才会响应------效率低）
       * Confirm 确认模式 和 return 退回模式

### Kafka

1. Partition Leader 选举，为什么由 Controller 来完成，而不是由 follow 去竞争（争注册临时节点）？
   性能考虑。
   Controller 独裁： 优点：直接指定，高效。 缺点：复杂

   follow 民主竞争：缺点：选票过半性能低

## 分布式定时器方案

## Mysql

1. mysql io 模型
   MySQL网络IO采用了select/poll模型。
   Linux上默认用的是poll，而Windows上则是select。
   （poll模型主要解决了select的最大文件描述符限制，但不具备移植性，在Linux上有实现，Windows没有。）
   参考：[《MySQL系列（5）：mysqld之网络IO模型》](https://blog.csdn.net/CanvaChen/article/details/103847742?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-1-103847742.pc_agg_new_rank&utm_term=mysql+io%E6%A8%A1%E5%9E%8B&spm=1000.2123.3001.4430)

## Redis

1. Redis为什么使用skiplist而不是平衡树

   * 内存方面
     skiplist并不是特别耗内存，只需要调整下节点到更高level的概率，就可以做到比B树更少的内存消耗。
   * 范围查询
     sorted set可能会面对大量的zrange和zreverange操作，跳表作为单链表遍历的实现性能不亚于其他的平衡树。
   * 实现和调试起来比较简单。 
     例如，实现O(log(N))时间复杂度的ZRANK只需要简单修改下代码即可

   参考：[《Redis源码剖析之跳表(skiplist)》](https://juejin.cn/post/6921608018463293447)

2. 跳表替换红黑树（为什么不把所有红黑树的场景都替换成跳表）
   跳表更占用内存

3. 跳表找第几名
   node 间间隔