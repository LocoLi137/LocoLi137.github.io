---
title: 生产经验-kafka
date: 2023-09-07 15:21:40
tags:
  - '消息队列'
  - '生产经验'
categories:
  - '消息队列'
---
# kafka

# 生产者提升吞吐量

```.properties
#批次大小,默认16k
batch.size:
#等待时间,修改为5-100ms
lingers.ms:
#压缩snappy
compression.type:
#缓冲区大小，修改为64m
RecordAccumulator:
```

# 数据可靠性

### 数据发送流程

![](https://article.biliimg.com/bfs/article/0fba7e0af2ba4c30ebbe764a362b2aeaf8584835.png)

### ack应答

ACK应答级别

- 0：生产者发送过来的数据，不需要等数据落盘，直接应答

  ![](https://article.biliimg.com/bfs/article/be8ac8685cb023701a5f03e0d174094a263fb005.png)

- 1：生产者发送过来的数据，Leader收到后应答(不等follower)

  ![](https://article.biliimg.com/bfs/article/4899078cff0316b080d994df39877348afdf3387.png)

- -1：生产者发送过来的数据，Leader和ISR队列里面的所有节点收齐数据后应答。

  ![](https://article.biliimg.com/bfs/article/b7bdf943512985294b288f58ce43aa549d7badae.png)

**问题**：Leader收到数据，所有Follower都开始同步数据，但有一个Follower，因为某种故障不能与Leader进行同步，导致Leader无法回复

Leader维护了一个动态的`in-sync replica set(ISR)`，意为和Leader保持同步的Follower+Leader集合(leader：0，isr:0,1,2)。如果Follower长时间未向Leader发送通信请求或同步数据，则该Follower将被踢出ISR。该时间阈值由`replica.lag.time.max.ms`参数设定，默认30s。例如2超时，(leader:0, isr:0,1)。这样就不用等长期联系不上或者已经故障的节点。

**数据可靠性分析**：如果分区副本设置为1个，或者ISR里应答的最小副本数量(`min.insync.replicas` 默认为1)设置为1，和ack=1的效果是一样的，仍然有丢数的风险（leader：0，isr:0）。

**数据完全可靠条件** = ACK set -1 + 分区副本≥2 + ISR应答最小副本数量≥2

**可靠性总结**：

-   acks=0，生产者发送过来数据就不管了，可靠性差，效率高；
-   acks=1，生产者发送过来数据Leader应答，可靠性中等，效率中等；
-   acks=-1，生产者发送过来数据Leader和ISR队列里面所有Follwer应答，可靠性高，效率低；

在生产环境中，acks=0很少使用；acks=1，一般用于传输普通日志，允许丢个别数据；acks=-1，一般用于传输和moneyyyyyyyyyyyyyyyyyyy相关或其他重要数据，对可靠性要求贼高的场景。

# 数据去重

### 数据传递语义&#xD;

-   至少一次（At Least Once）= ACK set -1 + 分区副本≥2 + ISR应答最小副本数量≥2
-   最多一次（At Most Once）= ACK级别设置为0
-   总结：
    At Least Once可以保证数据不丢失，但是不能保证数据不**重复**；
    At Most Once可以保证数据不重复，但是不能保证数据不**丢失**。
-   精确一次（Exactly Once）：对于一些非常重要的信息，比如和钱相关的数据，要求数据**既不能重复也不丢失**。
    Kafka 0.11版本以后，引入了一项重大特性：幂等性和事务。

### 幂等性

开启参数 `enable.idempotence` 默认为 true，false 关闭。

幂等性指Producer不论向Broker发送多少次重复数据，Broker端都只会持久化一条，保证了不重复。

**精确一次**(Exactly Once)=幂等性+至少一次（ack=-1 + (分区副本数>=2)+ISR最小副本数量>=2）

重复数据的判断标准：具有【PID, Partition, SeqNumber】相同主键的消息提交时，Broker只会持久化一条。其中PID是Kafka每次重启都会分配一个新的；Partition 表示分区号；Sequence Number是单调自增的。

所以幂等性**只能保证的是在单分区单会话内不重复**。

![](https://article.biliimg.com/bfs/article/7e2c54a7546581b4eda7ab72681925c59990c066.png)

### 生产者事务

说明：开启事务，必须开启幂等性。

**Kafka 事务原理**

![](https://article.biliimg.com/bfs/article/7e0e4a0458c316aafa6d5469bc420fd92ca53619.png)

**API**

```java
// 1 初始化事务
void initTransactions();
// 2 开启事务
void beginTransaction() throws ProducerFencedException;
// 3 在事务内提交已经消费的偏移量（主要用于消费者）
void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets, String consumerGroupId) throws ProducerFencedException;
// 4 提交事务
void commitTransaction() throws ProducerFencedException;
// 5 放弃事务（类似于回滚事务的操作）
void abortTransaction() throws ProducerFencedException;
```

# 数据有序

![](https://article.biliimg.com/bfs/article/7f17f82e3479be11ca8b359a1374cdd2ee998083.png)

# 数据乱序

-   kafka在1.x版本之前保证数据单分区有序，条件如下：
    `max.in.flight.requests.per.connection=1`（不需要考虑是否开启幂等性）。
-   kafka在1.x及以后版本保证数据单分区有序，条件如下：
    -   开启幂等性
        `max.in.flight.requests.per.connection`需要设置小于等于5。
    -   未开启幂等性
        `max.in.flight.requests.per.connection`需要设置为1。
    -   原因说明：因为在kafka1.x以后，启用幂等后，kafka服务端会缓存producer发来的最近5个request的元数据，
        故无论如何，都可以保证最近5个request的数据都是有序的。

![](https://article.biliimg.com/bfs/article/426e4608518f85f34cacbb963d3a9f0f7c0447b5.png)

![](https://article.biliimg.com/bfs/article/299b82f51be683d26b7f4fc048f63cd6a88a2ab1.png)

# 节点服役和退役

# 手动调整分区副本存储

# Leader Partition负载均衡

# 增加副本因子

# 分区的分配&再平衡

# 消费者事物

# 数据积压（消费者提升吞吐量）