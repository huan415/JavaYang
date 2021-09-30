# 记一次Kafka提交报错

## 背景

用 Canal+Kafka 做数据异构，对 kafka 客户端做了一层封装（没有用 springboot 封装好的注解）

```java
while (true) {
      List<KafkaFlatMessage> messages = connector.getFlatListWithoutAck(100L, TimeUnit.MILLISECONDS, offset);
}
```



报错如下：

```java
2021-09-14 10:37:00.065 [Thread-26] ERROR c.l.c.canaltool.kafka.CanalKafkaOffsetClientFlat [] - Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing max.poll.interval.ms or by reducing the maximum size of batches returned in poll() with max.poll.records.
org.apache.kafka.clients.consumer.CommitFailedException: Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing max.poll.interval.ms or by reducing the maximum size of batches returned in poll() with max.poll.records.
        at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:900)
        at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:840)
        at org.apache.kafka.clients.consumer.internals.AbstractCoordinator$CoordinatorResponseHandler.onSuccess(AbstractCoordinator.java:978)
        at org.apache.kafka.clients.consumer.internals.AbstractCoordinator$CoordinatorResponseHandler.onSuccess(AbstractCoordinator.java:958)
        at org.apache.kafka.clients.consumer.internals.RequestFuture$1.onSuccess(RequestFuture.java:204)
        at org.apache.kafka.clients.consumer.internals.RequestFuture.fireSuccess(RequestFuture.java:167)
        at org.apache.kafka.clients.consumer.internals.RequestFuture.complete(RequestFuture.java:127)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient$RequestFutureCompletionHandler.fireCompletion(ConsumerNetworkClient.java:578)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.firePendingCompletedRequests(ConsumerNetworkClient.java:388)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:294)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:233)
        at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:212)
        at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator.commitOffsetsSync(ConsumerCoordinator.java:693)
        at org.apache.kafka.clients.consumer.KafkaConsumer.commitSync(KafkaConsumer.java:1368)
        at org.apache.kafka.clients.consumer.KafkaConsumer.commitSync(KafkaConsumer.java:1330)
        at com.alibaba.otter.canal.client.kafka.KafkaCanalConnector.ack(KafkaCanalConnector.java:274)

```

![](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/kafka_commit_fail_interval_ms_1.jpg)

![](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/kafka_commit_fail_interval_ms_2.jpg)



## 原因

1. rebalanced ？ commit 之前有发生过 rebalanced ？为什么会发送 rebalanced ？ max.poll.interval.ms 又是啥？
   原因：心跳超时。消费者在消费消息时间太长。在 KafkaCanalConnector 中设置了 session.timeout.ms = 30s ，而上面日志报错可以看出间隔时间 > 30s （拉完消息给业务处理，处理完再拉下一批），所以超时，误以为 Consumer 挂了，就发生了 Rebalanced 。

## 知识点解析

1. session.timeout.ms：超时时间。

   作用：允许 Consumer 失联的最大时间
   原理：在该值的时间内，没有收到 Consumer 请求，就认为 Consumer 挂了，从 group 中移除。

2. heartbeat.interval.ms：发送心跳间隔时间
   作用：Consumer 加入或离开 group 时促进 Reblance
   原理：必须小于session.timeout.ms的1/3。该值越小，Replance时间越小

3. max.poll.interval.ms：拉取消息的间隔时间
   作用：Consumer pull 超时或者失败
   原理：Consumer 两次 pull 的时间超过这个值，就认为消费组能力不足，**此时commit失败**，并从group移除，触发 Replance

证明 Consumer 正常的两种方式：

1. 心跳：heartbeat.interval.ms 周期性发送心跳，超过 session.timeout.ms 认为 Consumer 不可用。
2. pull 间隔时间：max.poll.interval.ms 时间内，没有进行 pull 就认为服务不可用


参考： [不会写代码的dba不是一个好运维](https://www.cnblogs.com/iamwangjingwei/)