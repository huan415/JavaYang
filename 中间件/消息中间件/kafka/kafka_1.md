# kafka_1

## 概念

### Broker

​        即：kafka节点。多个Broker组成kafka集群。

### Producer

​        生产者

### Consumer

​        消费组
​        consumer自己维护自己的offset

### ConsumerGroup

​        消费者组

### Topic

​        主题（逻辑概念）

### Partition

​        分区（物理概念）
​        有序的message序列。每个partition，对应一个commit log文件
​        分区目的：1. 提供并发
​                          2.commit log受机器资源限制，可以分在不同的机器上

### 副本

1. 几个名称解释
   * leader： 负责partition的读写
   * replicas:  所有发副本，包括存活的和挂掉的
   * ISR：当前存活的副本



## 注意点

* 同一个partition保证顺序消费，同一个topic里不同的partition不保证顺序消费
* ConsumerGroup里的consumer不要比partition，因为多出来的consumer消费不到消息





## 常用命令

```shell
#查看主题
./kafka‐topics.sh ‐‐list ‐‐zookeeper localhost:2181
#创建主题
./kafka‐topics.sh ‐‐create ‐‐zookeeper localhost:2181 ‐‐replication‐factor 1 ‐‐partitions 1 ‐‐topic test
#删除主题
./kafka‐topics.sh ‐‐delete ‐‐topic test ‐‐zookeeper localhost:2181
#修改主题
./kafka-topics.sh --zookeeper localhost:2181 --alter --topic delete --partitions 1


#发送消息
./kafka‐console‐producer.sh ‐‐broker‐list localhost:9092 ‐‐topic test
#消费主题
./kafka‐console‐consumer.sh ‐‐bootstrap‐server localhost:9092 ‐‐consumer‐property group.id=testGroup ‐‐topic test
#从头开始消费
./kafka‐console‐consumer.sh ‐‐bootstrap‐server localhost:9092 ‐‐from‐beginning ‐‐topic test
#消费多主题
./kafka‐console‐consumer.sh ‐‐bootstrap‐server localhost:9092 ‐‐whitelist "test|test1"


#查看消费者组
./kafka‐consumer‐groups.sh ‐‐bootstrap‐server localhost:9092 ‐‐list
#查看消费者偏移量
./afka‐consumer‐groups.sh ‐‐bootstrap‐server localhost:9092 ‐‐describe ‐‐group testGroup




```

