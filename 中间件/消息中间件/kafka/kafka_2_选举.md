# Kafka选举

## Kafka核心控制器（Kafka Controller）选举

1. **概念**
   Kafka集群有多个broker，会选举一个broker为控制器（Kafka Controller），负责管理集群中所有的分区和副本。

   * 某个分区的leader副本发送故障，控制器负责选举新的leader副本
   * 检测到某个分区的ISR集合变化，控制器负责通知所有broker更新元素据信息
   * 分区数发生变化时，控制器负责partition的重新分配

2. **Kafka核心控制器的选举过程**
   集群中每个broker尝试在zookeeper中创建一个/controller, 但是只会有一个broker能创建成功，创建成功就成为controller
   当controller宕机，zookeeper临时节点消息，集群中其他broker一直都在监听这个临时节点，发现临时节点消失，会竞争创建这个临时节点。
   流程：

   1. 监听Broker变化。在Zookeeper中/brokers/ids/节点底下，添加BrokerChangeListener，用来监听broker的增减

   2. 监听topic变化。在Zookeeper中/brokers/topics/节点底下，添加TopicChangeListener，用来监听topic的变化

      ​                          在Zookeeper中/admin/delete_topics/节点底下，添加TopicDeletionListener，用来监听topic的删除
      
   3. 从Zookeeper中读取当前所有borker、topic、partition等信息，并进行管理。
      在Zookeeper中/brokers/topics/[topic]节点底下，添加PartitionModificationsListener，用来监听topic的分区变化
   
   4. 更新元素据到其他broker节点

## Partition副本Leader选举

1. 顺序选replicas中的broker,  且该broker在ISR中

2. 概念：controller监听了broker的变化（BrokerChangeListener），当分区leader所在的broker挂断，controller从partition的replicas副本中选出第一个broker作为leader(这个broker必须在ISR列表中)

   

## 组协调器选举（GroupCoordinator）

每个consumer group都会选择一个broker作为自己的主协调器coordinator,负责监控组内所有consumer的心跳。如果consumer宕机触发reblance机制。

公式：hash(consumer groupId) % topic的分区数。算出offset提交到哪个分区，这个分区的leader所在的broker就是这个消费者组的GroupCoordinator
(每个分区有多个副本，其中会有一个副本leader，即：每个分区都会有一个leader)4



## 消费者Leader选举

第一个发送消息给GroupCoordinator的consumer被选举为leader

目的：制定分区方案，制定完方案后发给GroupCoordinator，由GroupCoordinator下发给其他消费者（因为consumer之间无法直接通信）

## Consumer如何记录offset

1. Consumer会定期将offset记录到kafka内部主题（_consumer_offsets，offsets.topic.num.partitions设置分区数，默认50个分区）
   key: consumerGroupId + topic + 分区号
   value: offset值

## Reblance机制

1. consumerGroup中某个consumer挂掉，会将他负责的分区交给其他消费者。
2. 什么时候会触发Reblance机制
   * consumer增加或者减少
   * topic分区数增加或者减少
   * 消费者组订阅更多的topic
3. 流程
   * 选择组协调器
     consumer启动时，会向broker发送FindCoordinatorRequest，获取组协调器GroupCoordinator并建立连接
   * 加入消费者（Join Group）
     consumer向 GroupCoordinator发送 JoinGroupRequest请求(第一个加入组织的consumer作为leader)。GroupCoordinator将这个consumer的消息告诉leader，leader负责制定分区方案
   * （SYNC GROUP）
     leader给GroupCoordinator发送SyncGroupRequest，
     GroupCoordinator就把分区方案下发给各 个consumer，他们会根据指定分区的leader broker进行网络连接以及消息消费

## Reblance策略

* range策略（按序号排序）----默认
  n = 分区数/消费者数量，m=分区数%消费者数
  前m个消费者：n+1个分区，后面剩余的消费者：n个分区
* round-rodin策略（轮询）
  轮询
* sticky策略（当一个消费者挂掉，将其平均分配给其他消费者，即：其他消费者原有的分区不变）
  两个原则：
  * 分配尽可能均匀
  * 分配尽可能与上次保存一致

## Producer发布消息

1. 写入方式
   producer push消息到broker，broker将消息append到partition（顺序写，提交kafka吞吐量）
2. 消息路由（路由到哪个partition）
   * 有指定partiton，直接发到那个partition
   * 未指定partition，但有指定key，对key进行hash选出partition
   * 未指定partition，未指定key，轮询选出partition
3. 写入流程
   1. producer从Zookeeper的/brokers/..state节点找出leader
   2. producer将消息发给leader
   3. leader将消息写到log
   4. 同步副本消息：followers从leader pull消息，写入本地log后发送ACK
   5. leader收到ISR中的replica的ACK后，增加HW（消费者能消费的最高水位），然后向producer发送ACK

## HW 与 LEO

* HW高水位（HighWatermark）（消费者能消费的最高水位）
  ISR中最小的LEO 即为HW
  保证producer发送消息成功：leader写入消息不能马上被消息，要等到leader将消息同步到ISR中各个副本，后更新HW，这时才能被消费者消费
* LEO  (log-end-offsets)
