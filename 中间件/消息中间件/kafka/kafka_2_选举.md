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

