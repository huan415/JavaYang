# Rabbitmq概念

## 生产者

## 消费组

## 队列（Queue）

## 交换器（Exchange）

1. Producer发送消息后，经Exchange路由到各个队列中（路由不到则抛弃）
2. 类型
   * direct
     交换器发送消息：BindingKey 和 Routingkey完全匹配的队列
   * fanout
     交换器发送消息：所有与其绑定的队列
   * topic
     交换器发送消息：BindingKey 和 Routingkey 模糊规则匹配
   * headers
     交换器发送消息：不依赖Routingkey，由headers模糊规则匹配

## 路由键（RoutingKey）

1. 生产者将消息发给交换器后，由RoutingKey决定消息流向哪儿

## 绑定（Binding）

1. 将Exchange与Queue绑定起来（绑定时指定RoutingKey）



## Rabbitmq六种工作模式

1. Simple简单模式
2. Work工作模式
3. publish/subscribe模式 (发布订阅模式)
4. routing路由模式
5. topic主题模式
6. 

## 流转过程

1. Producer连接Broker，建立连接(Connection)，开启信道(Channel)
2. 声明交换器
3. 声明队列
4. 通过路由键和队列绑定起来
5. Producer发送消息到Broker（指定Exchange、RoutingKey）
6. 