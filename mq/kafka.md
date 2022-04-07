# kafka高性能原理

[Kafka如何实现高性能IO？](https://time.geekbang.org/column/article/126493)

- 使用批量消息提升服务端处理能力

  在 Kafka 的客户端 SDK（软件开发工具包）中，Kafka 的 Producer 只提供了单条发送的 send() 方法，并没有提供任何批量发送的接口。原因是，Kafka 根本就没有提供单条发送的功能，是的，你没有看错，虽然它提供的 API 每次只能发送一条消息，但实际上，Kafka 的客户端 SDK 在实现消息发送逻辑的时候，采用了异步批量发送的机制。

  在服务端，Kafka 不会把一批消息再还原成多条消息，再一条一条地处理，这样太慢了。Kafka 这块儿处理的非常聪明，每批消息都会被当做一个“批消息”来处理。也就是说，在 Broker 整个处理流程中，无论是写入磁盘、从磁盘读出来、还是复制到其他副本这些流程中，批消息都不会被解开，一直是作为一条“批消息”来进行处理的。

  构建批消息和解开批消息分别在发送端和消费端的客户端完成，不仅减轻了 Broker 的压力，**最重要的是减少了 Broker 处理请求的次数**，提升了总体的处理能力。

- 使用顺序读写提升磁盘 IO 性能

  对于每个分区，它把从 Producer 收到的消息，顺序地写入对应的 log 文件中，一个文件写满了，就开启一个新的文件这样顺序写下去。消费的时候，也是从某个全局的位置开始，也就是某一个 log 文件中的某个位置开始，顺序地把消息读出来。

- 利用 PageCache 加速消息读写

  Kafka 在读写消息文件的时候，充分利用了 PageCache 的特性。一般来说，消息刚刚写入到服务端就会被消费，按照 LRU 的“优先清除最近最少使用的页”这种策略，读取的时候，对于这种刚刚写入的 PageCache，命中的几率会非常高。

  也就是说，大部分情况下，消费读消息都会命中 PageCache，带来的好处有两个：一个是读取的速度会非常快，另外一个是，给写入消息让出磁盘的 IO 资源，间接也提升了写入的性能。

- ZeroCopy：零拷贝技术

  Kafka 的代码调用了 Java NIO 库，具体是 FileChannel 里面的 transferTo 方法

# Rebalance

> [消费者组重平衡能避免吗？](https://time.geekbang.org/column/article/105737)

Rebalance 就是让一个 Consumer Group 下所有的 Consumer 实例就如何消费订阅主题的所有分区达成共识的过程。在 Rebalance 过程中，所有 Consumer 实例共同参与，在协调者组件的帮助下，完成订阅主题分区的分配。但是，**在整个过程中，该消费者组所有实例都不能消费任何消息，因此它对 Consumer 的 TPS 影响很大**。

所谓协调者，在 Kafka 中对应的术语是**Coordinator**，它专门为 Consumer Group 服务，负责为 Group 执行 Rebalance 以及提供位移管理和组成员管理等。具体来讲，**Consumer 端应用程序在提交位移时，其实是向 Coordinator 所在的 Broker 提交位移**。同样地，当 Consumer 应用启动时，也是向 Coordinator 所在的 Broker 发送各种请求，然后由 Coordinator 负责执行消费者组的注册、成员管理记录等元数据管理操作。

**所有 Broker 在启动时，都会创建和开启相应的 Coordinator 组件**。也就是说，所有 Broker 都有各自的 Coordinator 组件。那么，Consumer Group 如何确定为它服务的 Coordinator 在哪台 Broker 上呢？答案就在我们之前说过的 Kafka 内部位移主题 __consumer_offsets 身上。

1. 确定**由位移主题的哪个分区来保存该 Group 数据**：partitionId=Math.abs(groupId.hashCode() % offsetsTopicPartitionCount)。
2. 找出该分区 Leader 副本所在的 Broker，该 Broker 即为对应的 Coordinator

Rebalance 的弊端总结起来有以下 3 点：

1. Rebalance 影响 Consumer 端 TPS。这个之前也反复提到了，这里就不再具体讲了。总之就是，在 Rebalance 期间，Consumer 会停下手头的事情，什么也干不了。
2. Rebalance 很慢。如果你的 Group 下成员很多，就一定会有这样的痛点。还记得我曾经举过的那个国外用户的例子吧？他的 Group 下有几百个 Consumer 实例，Rebalance 一次要几个小时。在那种场景下，Consumer Group 的 Rebalance 已经完全失控了。
3. Rebalance 效率不高。当前 Kafka 的设计机制决定了每次 Rebalance 时，Group 下的所有成员都要参与进来，而且通常不会考虑局部性原理，但局部性原理对提升系统性能是特别重要的。

## 如何避免 Rebalance

rebalance 发生的时机有三个：

1. 组成员数量发生变化订阅
2. 主题数量发生变化
3. 订阅主题的分区数发生变化

后面两个通常都是运维的主动操作，所以它们引发的 Rebalance 大都是不可避免的。接下来，我们主要说说因为组成员数量变化而引发的 Rebalance 该如何避免。

我们更在意的是 Group 下实例数减少这件事。如果你就是要停掉某些 Consumer 实例，那自不必说，关键是在某些情况下，Consumer 实例会被 Coordinator 错误地认为“已停止”从而被“踢出”Group。如果是这个原因导致的 Rebalance，我们就不能不管了。

Coordinator 会在什么情况下认为某个 Consumer 实例已挂从而要退组呢？

1. 当 Consumer Group 完成 Rebalance 之后，**每个 Consumer 实例都会定期地向 Coordinator 发送心跳请求**，表明它还存活着。如果某个 Consumer 实例不能及时地发送这些心跳请求，Coordinator 就会认为该 Consumer 已经“死”了，从而将其从 Group 中移除，然后开启新一轮 Rebalance。Consumer 端有个参数，叫 **session.timeout.ms**，就是被用来表征此事的。该参数的默认值是 10 秒，即如果 Coordinator 在 10 秒之内没有收到 Group 下某 Consumer 实例的心跳，它就会认为这个 Consumer 实例已经挂了。可以这么说，session.timeout.ms 决定了 Consumer 存活性的时间间隔。
2. 除了这个参数，Consumer 还提供了一个允许你**控制发送心跳请求频率的参数，就是 heartbeat.interval.ms**。这个值设置得越小，Consumer 实例发送心跳请求的频率就越高。频繁地发送心跳请求会额外消耗带宽资源，但好处是能够更加快速地知晓当前是否开启 Rebalance，因为，目前 Coordinator 通知各个 Consumer 实例开启 Rebalance 的方法，就是将 REBALANCE_NEEDED 标志封装进心跳请求的响应体中
3. Consumer 端还有一个参数，用于控制 Consumer 实际消费能力对 Rebalance 的影响，即 **max.poll.interval.ms 参数。它限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔**。它的默认值是 5 分钟，表示你的 Consumer 程序如果在 5 分钟之内无法消费完 poll 方法返回的消息，那么 Consumer 会**主动发起“离开组”的请求**，Coordinator 也会开启新一轮 Rebalance。

# CommitFailedException异常怎么处理？

[CommitFailedException异常怎么处理？](https://time.geekbang.org/column/article/107845)

看看社区对这个异常的最新解释：

```
Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing max.poll.interval.ms or by reducing the maximum size of batches returned in poll() with max.poll.records.
```

> 这段话前半部分的意思是，本次提交位移失败了，原因是消费者组已经开启了 Rebalance 过程，并且将要提交位移的分区分配给了另一个消费者实例。出现这个情况的原因是，你的消费者实例连续两次调用 poll 方法的时间间隔超过了期望的 max.poll.interval.ms 参数值。这通常表明，你的消费者实例花费了太长的时间进行消息处理，耽误了调用 poll 方法。

解决方案：

- 缩短单条消息处理的时间。比如，之前下游系统消费一条消息的时间是 100 毫秒，优化之后成功地下降到 50 毫秒，那么此时 Consumer 端的 TPS 就提升了一倍。
- 增加 Consumer 端允许下游系统消费一批消息的最大时长。这取决于 Consumer 端参数 max.poll.interval.ms 的值。在最新版的 Kafka 中，该参数的默认值是 5 分钟。如果你的消费逻辑不能简化，那么提高该参数值是一个不错的办法。值得一提的是，Kafka 0.10.1.0 之前的版本是没有这个参数的，因此如果你依然在使用 0.10.1.0 之前的客户端 API，那么你需要增加 session.timeout.ms 参数的值。不幸的是，session.timeout.ms 参数还有其他的含义，因此增加该参数的值可能会有其他方面的“不良影响”，这也是社区在 0.10.1.0 版本引入 max.poll.interval.ms 参数，将这部分含义从 session.timeout.ms 中剥离出来的原因之一。
- 减少下游系统一次性消费的消息总数。这取决于 Consumer 端参数 max.poll.records 的值。当前该参数的默认值是 500 条，表明调用一次 KafkaConsumer.poll 方法，最多返回 500 条消息。可以说，该参数规定了单次 poll 方法能够返回的消息总数的上限。如果前两种方法对你都不适用的话，降低此参数值是避免 CommitFailedException 异常最简单的手段。
- 下游系统使用多线程来加速消费。

# 在 Kafka 中，ZooKeeper 的作用是什么？

> 目前，Kafka 使用 ZooKeeper 存放集群元数据、成员管理、Controller 选举，以及其他一些管理类任务。
>
> “存放元数据”是指主题分区的所有数据都保存在 ZooKeeper 中，且以它保存的数据为权威，其他“人”都要与它保持对齐。
>
> “成员管理”是指 Broker 节点的注册、注销以及属性变更，等等。
>
> “Controller 选举”是指选举集群 Controller，而其他管理类任务包括但不限于主题删除、参数配置等。

# 规划磁盘容量

Kafka 集群到底需要多大的存储空间？这是一个非常经典的规划问题。Kafka 需要将消息保存在底层的磁盘上，这些消息默认会被保存一段时间然后自动被删除。虽然这段时间是可以配置的，但你应该如何结合自身业务场景和存储需求来规划 Kafka 集群的存储容量呢？

我举一个简单的例子来说明该如何思考这个问题。假设你所在公司有个业务每天需要向 Kafka 集群发送 1 亿条消息，每条消息保存两份以防止数据丢失，另外消息默认保存两周时间。现在假设消息的平均大小是 1KB，那么你能说出你的 Kafka 集群需要为这个业务预留多少磁盘空间吗？

我们来计算一下：每天 1 亿条 1KB 大小的消息，保存两份且留存两周的时间，那么总的空间大小就等于 1 亿 * 1KB * 2 / 1000 / 1000 = 200GB。一般情况下 Kafka 集群除了消息数据还有其他类型的数据，比如索引数据等，故我们再为这些数据预留出 10% 的磁盘空间，因此总的存储容量就是 220GB。既然要保存两周，那么整体容量即为 220GB * 14，大约 3TB 左右。Kafka 支持数据的压缩，假设压缩比是 0.75，那么最后你需要规划的存储空间就是 0.75 * 3 = 2.25TB。总之在规划磁盘容量时你需要考虑下面这几个元素：

- 新增消息数
- 消息留存时间
- 平均消息大小
- 备份数
- 是否启用压缩