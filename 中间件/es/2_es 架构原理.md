# ES 架构原理

## 节点

1. Master 节点
   es 启动时，会跟其他节点建立连接，并从候选节点中选举一个作为主节点

   Master 节点负责：

   1. 创建、删除索引；分片
   2. 元数据
   3. 集群节点状态
   4. **不负责数据的写入或查询（压力小）**

2. DataNode 节点
   DataNode 节点负责：

   1. **数据的写入或查询（压力大）**



## ES 集群概念

分布式搜索引擎：索引的数据分成若干部分，分布在不同的服务器节点中。

1. 分片：不同服务器节点中的索引数据。
   一个索引(index) 分成多个分片（shard）
2. 副本：每个分片都会有对应的副本（主分片、副本分片）
   （容错：避免某个节点宕机，导致整个分片都不可用）

```json
PUT /es_db
{
    XXX
    "settings":{
       "number_of_shards":3,
       "number_of_replicas":2
    }
}
```

