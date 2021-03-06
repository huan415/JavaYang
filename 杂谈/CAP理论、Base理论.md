# CAP理论、Base理论

## CAP理论

Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性）

- 一致性（C）：在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）
  俗话：例如kafka副本等情况，同步需要时间，当还没来得急同步就挂掉，数据不一致。
- 可用性（A）：在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）
  俗话：集群时，一个节点挂掉，其他节点还能提供服务
- 分区容忍性（P）：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。
  俗话：有很多机房，例如：上海机房、杭州机房，其中一个机房挂掉，其他机房可以正常工作。

### CAP只能满足其中两个

AP:  可用性、分区容忍性，放弃一致性
        不用等到所有节点的数据都同步完，就可以对外提供服务
         例如：Eureka

CP:  一致性、分区容忍性，放弃可用性
        为了一致性，要等所有数据都同步之后才对外提供服务，一段时间内不能用，所以牺牲了可用性
        例如：Zookeeper、Consul

AC:  一致性、可用性，放弃分区容忍性
        放弃分区容忍性, 意味着放弃系统的扩展性，违背分布式系统的涉及初衷
        例如：MySQL



## Base理论

BASE是对CAP中一致性和可用性权衡的结果，基于CAP定理逐步演化而来

Basically Available（基本可用）、Soft state（软状态）和Eventually consistent（最终一致性）



- 基本可用性：出现故障时，允许损失部分可用性，即保证核心可用
- 软状态：允许出现的存在中间状态，不会影响系统整体可用性
- 最终一致性：数据最终能够达到一致的状态



参考：https://blog.csdn.net/qq_17555933/article/details/104257222

​           https://www.cnblogs.com/duanxz/p/5229352.html
