---
title: kafka总结
tags: 
    - kafka
categories: 
    - kafka
---



# kafka总结

kafka是一款基于发布与订阅的消息系统，它是被设计用来解决日志收集、用户行为追踪等海量数据传输而设计的，kafka也通常被称为“分布式提交日志”或者“分布式流式平台”。



### kafka 架构图

![rebase](./kafka架构图.png)



### broker

broker代表了kafka的节点，一个节点对应一个进程，一个物理机或者虚拟机可以由一个或多个broker组成，多个broker配置不同的`broker_id`后可组成一个kafka集群，broker负责基础的消息写入、读取。



### controller

broker有一种叫做controller类型，controller除了负责broker基础功能外，还负责分区leader的选举，controller基于zookeeper临时节点的方式来确定，当第一个broker启动后，再zk上注册一个`/controller`目录让自己成为controller，其他节点则watch此临时节点，当broker宕机，`/controller`节点消失，其他watch的broker尝试注册/controller，注册成功的broker为下一个controller.

当集群中，controller感知到任意非自己的broker离线后（通过zk的broker注册信息感知），则需要为以当前offline-broker为分区leader的选取新的leader（分区leader选举通过ISR集合完成）



### Topic

kafka通过topic对消息做分类，主题就好比消息中间件的队列或者数据库的表，kafka的topic信息全部顺序追加写入到磁盘文件上



### Partition

主题的分区，一个topic至少有一个partition，可以在创建时指定，消息写入topic不同的partition默认是轮训写入，也可在客户端指定写入目标的分区，kafka只保证在同一个partition上写入的消息有序

![img](http://kafka.apache.org/24/images/log_anatomy.png)

对于每一个topic，都存在一个`Partition Leader`，只有`partition leader`才会接受消息读取与写入的请求，如果partition leader所在节点宕机，topic存在数据复制的节点的ISR集合，则从ISR集合中选取一个数据副本作为new partition leader，否则将`No leader`异常



### Replicate

数据的复制，kafka高可用的重要特征之一，kafka的topic数据默认只有一个数据副本，创建topic时，可指定数据副本数量，但数量不可超过集群的总数，当打开数据副本后，partition leader写入的数据会被同步到各个数据副本，所有的数据副本构成一个ISR集合，此集合存储在zk上，当partition leader宕机后，controller从ISR中选择一个消息长度最接近的副本为new partition leader

数据副本在磁盘上持久化时间可按需配置，kafka会周期性的启动clear线程来清理log文件



### Consumer

消费者，topic partition的消费者，每个消费者按照group进行分组，消费者将不同group的offset记录在kafka内部的`__consumer_offsets`中（2.x以前记录在zk中，但是对zk的性能消耗很大）,同一个group，消费者不支持多线程并发消费

消费者每次根据`max.poll.records`和`fetch.max.bytes`来拉去一组消息消费，而不是一条一条消费

消费者消费消息分为自动提交和手动提交，可根据`enable.auto.commit`调整，如果为自动提交，客户端会周期性的对offset进行提交操作，默认为`5000ms`

配置项可参考：`org.apache.kafka.clients.consumer.ConsumerConfig`



### Group Coordinator

组协调器, broker用于处理每个消费组的offset commit 请求, 将消费组的offset记录到`__consumer_offsets`



### Producer

消息生产者，将消息发送到指定的topic，可并发的写入

> 默认的消息最大长度为1M，发送超出限制的消息，不报错，无感知，排错困难，需谨慎



### 消息的丢失问题

kafka的消息在写入broker时, 可能因为网络抖动等原因, 导致kafka消息写入失败

```java
producer.send(); // 请求成功到达broker, 接收响应的时候, 网络抖动, 导致客户端认为消息发送失败
```

对于此问题, kafka有三种策略来处理(或者说是分布式系统有3中策略来保证)

* at-least-once 

  此策略表示至少一次接收到成功的ack响应. 假设producer发送了一条消息到broker, 由于超时导致了producer接收到了一个错误的ack, 此时producer将根据重试配置`request.required.acks`来进行重试, 如果此重试次数范围内还未收到正确的ack, 则认为消息发送失败. 
  消息被producer投递后, broker写入成功, 但是网络抖动后, 导致producer认为消息被投递失败了, 触发了重试机制, 那么此时, 消息将会被投递两次, 导致消费者消费到两条重复的消息, 进而违背了幂等性的约束.

  上述的失败也可能会导致消息在同一分区的乱序, producer默认一次可发起5条消息, 这5个消息之间无需等待上一个消息是否成功, 当第一条消息发送完成, 得到一个失败的ack, 重试发送第二次的时候, 第二条消息已经发送成功, 最终导致第一条消息和第二条消息的乱序.

* at-most-once 

  此策略表示最多一次接收到ack响应, 不管ack成功与否, producer将不再对结果进行处理, 此为kafka producer的默认行为

* exactly-once

  此策略表示确切的一次, 不管producer发送多少次, broker确切的只会接受一个消息, 此策略需要使用kafka的事务机制

> 上述所说的ack, 表示broker确认收到了消息的ack响应, 一般



### 一些遇到的坑

1. kafka broker日志中频繁出现topic的分区ISR的集合扩大缩小的日志

   ```less
   [2020-01-08 15:15:29,717] TRACE [Broker id=1] Cached leader info PartitionState(controllerEpoch=20, leader=4, leaderEpoch=58, isr=[4, 3], zkVersion=5071, replicas=[4, 2, 3], offlineReplicas=[]) for partition wifi-raw-data-0 in response to UpdateMetadata request sent by controller 2 epoch 20 with correlation id 365 (state.change.logger)
   [2020-01-08 15:15:37,220] TRACE [Broker id=1] Cached leader info PartitionState(controllerEpoch=20, leader=4, leaderEpoch=58, isr=[4, 3, 2], zkVersion=5072, replicas=[4, 2, 3], offlineReplicas=[]) for partition wifi-raw-data-0 in response to UpdateMetadata request sent by controller 2 epoch 20 with correlation id 366 (state.change.logger)
   ```

   此问题在topic打开了数据复制, 且数据量较大的时, 由于才副本落后于主副本过多, 或者说超过指定的时长未从主副本消费消息, kafka认为此时的从副本无法作为备份副本提供服务, 所以冲ISR集合中移除此副本信息, 可修改server.properties文件

   ```properties
   replica.lag.time.max.ms=30000
   ```

   > **replica.lag.time.max.ms**: If a follower hasn't sent any fetch  requests or hasn't consumed up to the leaders log end offset for at  least this time, the leader will remove the follower from isr
   >
   > - **Type**: long
   > - **Default**: 10000
   > - **Valid Values**: 
   > - **Importance**: high
   > - **Update Mode**: read-only

2. 低版本kafka升级高版本，导致broker日志大量zk连接超时问题，可在broker配置中放大zk超时配置

   ```properties
   zookeeper.connection.timeout.ms=60000
   zookeeper.session.timeout.ms=60000
   zookeeper.sync.time.ms=10000
   ```

   

3. 消费者频繁持续出现提交失败，错误信息提示超时，

   ```less
   Offset commit failed on partition fs-pos-filter-raw-data-0 at offset 4682179: The request timed out.
   ```

   部分说法是当前组所在的broker的组协调器有问题，组协调器无法正常提交offset到`__consumer_offsets`，要么删除 `__consumer_offsets`解决（删除后会立即创建，但不保证消息offset的正确性），要么更换消费组. 解决办法如下3种

   * 方法一

     删除`__consumer_offsets`, 让kafka集群重新自动构建

   * 方法二

     修改消费组, 使组协调器落到其他节点上

   * 方法三

     添加server.properties配置信息, 放大offset在集群内提交的超时时间

     ```properties
     offsets.commit.timeout.ms=30000
     ```

     > **offsets.commit.timeout.ms**: Offset commit will be delayed  until all replicas for the offsets topic receive the commit or this  timeout is reached. This is similar to the producer request timeout.
     >
     > - **Type**: int
     > - **Default**: 5000
     > - **Valid Values**: [1,...]
     > - **Importance**: high
     > - **Update Mode**: read-only

   * 方法四
     删除有问题的group:

     ```shell
     ./bin/kafka-consumer-groups \
         --bootstrap-server <bootstrap_server(s)> \
         --delete \
         --group <consumer_group_name> 
     ```

     

4. 日志持续输出错误信息：

   ```less
   2020-01-09 15:14:38.873 ERROR org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - [Consumer clientId=consumer-4, groupId=ls-seller-info-be] Offset commit failed on partition ls-seller-info-raw-data-0 at offset 11904: The request timed out.
   2020-01-09 15:14:38.873 INFO  org.apache.kafka.clients.consumer.internals.AbstractCoordinator - [Consumer clientId=consumer-4, groupId=ls-seller-info-be] Group coordinator qs-kfk-04:9092 (id: 2147483643 rack: null) is unavailable or invalid, will attempt rediscovery
   2020-01-09 15:14:38.974 INFO  org.apache.kafka.clients.consumer.internals.AbstractCoordinator - [Consumer clientId=consumer-4, groupId=ls-seller-info-be] Discovered group coordinator qs-kfk-04:9092 (id: 2147483643 rack: null)
   ```

   通过错误堆栈可以看到, 连续抛出3个异常错误日志, 第一个日志和2一致, 后续提示说组协调器无效, 然后重新发现了一个组协调器, 通过源码可知, 后续两个异常信息均由第一个错误引起的日志输出罢了.

   ConsumerCoordinator$OffsetCommitResponseHandler.java

   ```java
   /**
    * offset commit请求的响应处理 
   **/
   @Override
   public void handle(OffsetCommitResponse commitResponse, RequestFuture<Void> future) {
       sensors.commitLatency.record(response.requestLatencyMs());
       Set<String> unauthorizedTopics = new HashSet<>();
   
       for (OffsetCommitResponseData.OffsetCommitResponseTopic topic : commitResponse.data().topics()) {
         for (OffsetCommitResponseData.OffsetCommitResponsePartition partition : topic.partitions()) {
           TopicPartition tp = new TopicPartition(topic.name(), partition.partitionIndex());
           OffsetAndMetadata offsetAndMetadata = this.offsets.get(tp);
   
           long offset = offsetAndMetadata.offset();
   
           Errors error = Errors.forCode(partition.errorCode());
           if (error == Errors.NONE) {
             log.debug("Committed offset {} for partition {}", offset, tp);
           } else {
             if (error.exception() instanceof RetriableException) {
               log.warn("Offset commit failed on partition {} at offset {}: {}", tp, offset, error.message());
             } else {
               // group coordinator响应了一个错误信息, 超时. 对应到日志第一条
               log.error("Offset commit failed on partition {} at offset {}: {}", tp, offset, error.message());
             }
   
             if (error == Errors.GROUP_AUTHORIZATION_FAILED) {
               future.raise(new GroupAuthorizationException(groupId));
               return;
             } else if (error == Errors.TOPIC_AUTHORIZATION_FAILED) {
               unauthorizedTopics.add(tp.topic());
             } else if (error == Errors.OFFSET_METADATA_TOO_LARGE
                        || error == Errors.INVALID_COMMIT_OFFSET_SIZE) {
               // raise the error to the user
               future.raise(error);
               return;
             } else if (error == Errors.COORDINATOR_LOAD_IN_PROGRESS
                        || error == Errors.UNKNOWN_TOPIC_OR_PARTITION) {
               // just retry
               future.raise(error);
               return;
             } else if (error == Errors.COORDINATOR_NOT_AVAILABLE
                        || error == Errors.NOT_COORDINATOR
                        || error == Errors.REQUEST_TIMED_OUT) {
               // 此处满足条件Errors.REQUEST_TIMED_OUT,标记组协调器为未知状态, 并重新发现组协调器 
               markCoordinatorUnknown();
               future.raise(error);
               return;
             } else if (error == Errors.FENCED_INSTANCE_ID) {
               log.error("Received fatal exception: group.instance.id gets fenced");
               future.raise(error);
               return;
             } else if (error == Errors.UNKNOWN_MEMBER_ID
                        || error == Errors.ILLEGAL_GENERATION
                        || error == Errors.REBALANCE_IN_PROGRESS) {
               // need to re-join group
               resetGeneration();
               future.raise(new CommitFailedException());
               return;
             } else {
               future.raise(new KafkaException("Unexpected error in commit: " + error.message()));
               return;
             }
           }
         }
       }
   
       if (!unauthorizedTopics.isEmpty()) {
         log.error("Not authorized to commit to topics {}", unauthorizedTopics);
         future.raise(new TopicAuthorizationException(unauthorizedTopics));
       } else {
         future.complete(null);
       }
   }
   ```

   

5. storm kafkaSpout超时错误

   ```less
   org.apache.kafka.common.errors.TimeoutException: Timeout of 60000ms expired before successfully committing offsets {fs-pos-filter-raw-data-0=OffsetAndMetadata{offset=4560629, leaderEpoch=null, metadata='{"topologyId":"now-goods-hot-sale-2-1577371149","taskId":34,"threadName":"Thread-13-executor[34 34]"}'}}
   ```

   此问题频繁发生在kafka压力大的时候，Kafka client发出`OffsetCommitRequest`后, broker端的`Group Coordinator`将offset信息提交到`__consumer_offsets`的过程中时间过长, 导致kafka client端api调用超时

   > storm(1.2.3)中的KafkaSpout的Consumer采用的是手动同步提交的方式，默认的api调用的超时时间为60s

   目前处理的方式是修改了kafka client端api的超时时间, 60s修改为600s. 至于是什么原因导致的Group Coodinator提交offset信息到`__consumer_offsets`过慢, 未明确.

6. No Leader错误

   此错误一般由于topic为设置数据副本时, 分区leader所在在broker宕机后出现, 因为kafka默认只有分区leader才能提供服务, 只要对topic的分区做多副本容灾即可

   

