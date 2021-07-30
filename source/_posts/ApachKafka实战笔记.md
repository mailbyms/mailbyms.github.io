---
title: ApachKafka实战笔记
date: 2021-06-07 16:51:07
categories:
  - 笔记
tags:
  - kafka
---

- 在旧版本consumer 中，消费位移（ offset ）的保存与管理都是依托于ZooKeeper 来完成的。当数据量很大且消费很频繁时， ZooKeeper 的读／ 写性能往往容易成为系统瓶颈。这是旧版本consumer 为人话病的缺陷之一。而在新版本consumer 中，位移的管理与保存不再依靠 Zoo Keeper 了，自然这个瓶颈就消失了。
  - 可以从命令看出。用的是 --bootstrap-server
  - 位移不再保存在ZooKeeper 中，而是单独保存在Kafka 的一个内部topic 中： __consumer_offsets 
- kafka 消息留存时间：配置文件log.retention.hours=168，默认为 7 天
- Kafka producer 提供了一个默认的分区器。对于每条待发迭的消息而言，如果该消息指定了key ，那么该partitioner 会根据key 的哈希值来选择目标分区：若这条消息没有指定key ，则partitioner 使用轮询的方式确认目标分区
- 消息发送可能失败，例如分区的 leader 副本不可用（换届选举）。构建 producer 时指定 retrices 为 3，等待重试恢复。重试间隔（retry.backoff.ms）默认为100毫秒
- 一个 topic 在创建时可以指定多个分区，每个分区有多个副本。Consumer 组里的单个 comsumer 被指定负责某个分区的消息。
- ISR ，就是Kafka 集群动态维护的一组同步副本集合（ in-sync replicas ） 
  - 分区有多个副本（ replica ）
  - leader 副本对外提供服务，而其他副本被称为follower 副本，与leader保持同步
  - 落后leader 进度太多的follower 不能成为 leader 
- producer 的参数 acks：
  - 0: 发完一条马上发下一条
  - all 或者 -1: leader broker 写入本地，且等待 ISR 其它副本都写入本地后，才发响应结果
  - 1: leader broker 写入本地，就发响应结果
- 消费者组（consumer group）
  - 一个 consumer group 可能有若干个 consumer 实例（ 一个group 只有一个实例也是允许的）
  - 对于同一个group 而言， topic 的每条消息只能被发送到 group 下的一个 consumer 实例上
  - topic 消息可以被发送到多个 group 中
- 从Kafka consumer 的角度而言， poll 方法返回即认为consumer 成功消费了消息
- session.timeout.ms coordinator ：检测失败的时间” 。因此在实际使用中，用户可以为该参数设置一个比较小的值让coordinator 能够更快地检测consumer 崩溃的情况，从而更快地开启rebalance ，避免造成更
  大的消费滞后（ consumer lag ） 。目前该参数的默认值是10 秒。
- max.poll.interval.ms 
- max.poll.records 该参数控制单次poll 调用返回的最大消息数。默认值 500
- connections.max.idle.ms Kafka 会定期地关闭空闲Socket 连接导致下次consumer 处理请求时需要重新创建连向broker 的Socket 连接。当前默认值是9 分钟，如果用户实际环境中不在乎这些Socket 资源开销，比较推荐设置该参数值为 -1 ，即不要关闭这些空闲连接
- consumer 端需要为每个它要读取的分区保存消费进度，即分区中当前最新消费消息的位置。该位置就被称为位移（ offset ） 。consumer 需要定期地向Kafka 提交自己的位置信息。这里的位移值通常是下一条待消费的消息的位置。
- consumer 是自动提交位移的，自动提交间隔是5 秒，对应 auto.commit.interval.ms 参数
- coordinator 和 leader 
  - coordinator 通常是Kafka 集群中的一个broker ，组内所有consumer 向 coordinator 发送JoinGroup 请求。当收集全JoinGroup 请求后， coordinator 从中选择一个consumer 担任group 的leader
  - group 的 leader 是某个 consumer 实例，leader 负责为整个group 的所有成员制定分配方案
- KafkaConsumer 是非线程安全的，可选：
  - 每个线程维护一个KafkaConsumer
  - 单Kafka Consumer 实例 ＋多worker 线程
- KafkaProducer 是线程安全的，可以在多个线程中放心地使用同一个KafkaProducer 实例，事实上这也是社区推荐的producer 使用方法，因为通常它比每个线程维护一个Kafk:aProducer 实例效率要高
- coordinator 负责组管理工作，consumer （group）程序负责分区分配
