---
title: "RocketMQ 设计与理解"
date: 2021-04-05T11:22:38+08:00
draft: false
tags: ["rocketmq"]
---

## 整体架构

![rocketmqarchitecture1.png](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/rocketmq_architecture_1.png)

RocketMQ 主要由 Producer、Broker、Consumer、Name Server 四个部分组成。

其中Producer 负责生产消息，Consumer 负责消费消息，Broker 负责存储消息，Name server 充当路由消息的提供者。

每个 Broker 可以存储多个Topic的消息，每个Topic的消息也可以分片存储于不同的 Broker。Message Queue 用于存储消息的物理地址，每个Topic中的消息地址存储于多个 Message Queue 中。

![image-20210402173106318](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210402173106318.png?lastModify=1617587790)

## Topic

Topic 是一种逻辑上的分区，是同一类消息的集合，每一个消息只能属于一个 Topic ，是RocketMQ进行消息订阅的基本单位。

每个 topic 会被分成很多 Messsage Queue ，和 Kafka 中的 Partition 概念一样，topic 的数据被分布在不同的 Message Queue 中。

在业务增长，消息量增大时，可以增大 topic 的 Message Queue，这样可以将压力分摊到更多的 broker 上。因为 Producer 可以发送消息的时候可以通过指定的算法，将消息均匀的发送到每个 Message Queue。

## NameServer

生产者或消费者能够通过 Name Server查找各 Topic 相应的Broker IP 列表。 Name Server 可以多机部署变成一个集群保证高可用，但这些机器间彼此并不通信，也就是说三者的元数据舍弃了强一致性。

每一个 broker 启动时会向全部的 Name server 机器注册心跳，心跳里包含自己机器上 Topic 的拓扑信息，之后每隔 30s 更新一次，然后生产者和消费者启动的时候任选一台 Name Server 机器拉取所需的 Topic 的路由信息缓存在本地内存中，之后每隔 30s 定时从远端拉取更新本地缓存。

Name Server 机器中定时监测 broker 的心跳，一旦失联，即关闭这个 Broker 的连接，但是不主动通知生产组和消费组。也就是说对于 broker 的上线下线，需要 Producer 和 Consumer 主动去拉取才会跟新，因此二者最长需要 30s 才能感知到某个 broker 故障。

## Producer

Producer 负责生产消息，Producer 按照一定规则直接消息投递到 broker。RocketMQ 提供多种发送方式：

* 同步发送
* 异步发送
* 单向发送

同步和异步方式均需要Broker返回确认信息，单向发送不需要。

Producer 作为客户端发送消息时候，需要根据消息的 Topic 从本地缓存的 TopicPublishInfoTable 获取路由信息。如果没有则更新路由信息会从NameServer上重新拉取，同时Producer会默认每隔30s向NameServer拉取一次路由信息。

#### 负载均衡

Producer端在发送消息的时候，会先根据Topic找到指定的TopicPublishInfo，在获取了TopicPublishInfo路由信息后，RocketMQ的客户端在默认方式下从 TopicPublishInfo 中的 messageQueueList 中按照一定策略选择一个队列（MessageQueue）进行发送消息。

这个选择 MessageQueue，可以是随机选择，轮询，Hash 等，也支持自定义。

#### 高可用

* 发送重试
  RocketMQ 支持同步、异步发送，不管哪种方式都可以在配置失败后重试，如果单个 Broker 发生故障，重试会选择其他 Broker 保证消息正常发送。默认会尝试发送三次。
* 客户端容错
  RocketMQ  客户端会维护一个 Broker-发送延迟 关系。 根据这个关系选择一个发送延迟级别较低的 Broker 来发送消息，这样能最大限度地利用 Broker 的能力，剔除已经宕机、不可用或者发送延迟级别较高的Broker，尽量保证消息的正常发送。在选择时，对那些有“问题”的broker 进行退避。

## Consumer

Consumer 负责消费消息，一般是后台系统负责异步消费。一个消息消费者会从Broker服务器拉取消息、并将其提供给应用程序。

在使用消费者时需要指定一个组名。一个消费者组可以订阅多个Topic。一个消费者组订阅一个 Topic 的某一个 Tag，这种记录被称为订阅关系。RocketMQ规定消费订阅关系（消费者组名-Topic-Tag）必须一致，一个消费者组中的实例订阅的Topic和Tag必须完全一致，否则就是订阅关系不一致。订阅关系不一致会导致消费消息紊乱。

在Consumer启动后，它就会通过定时任务不断地向RocketMQ集群中的所有Broker实例发送心跳包（其中包含了，消息消费分组名称、订阅关系集合、消息通信模式和客户端id的值等信息）。

#### 消费模式

* 集群消费
  同一个消费组里的消费者“瓜分”所有消息来消费，同一个消息只会被一个消费者实例消费。因为集群模式的消费进度是保存在Broker端的，所以即使应用崩溃，消费进度也不会出错。
* 广播消费
  同一个消费组里的消费者会消费所有消息，即同一个消息会被每一个消费者实例消费。广播消费的消费进度保存在客户端机器的文件中。如果文件弄丢了，那么消费进度就丢失了，可能会导致部分消息没有消费。

#### 高可用

> consumer并不能配置从master读还是slave读。当master不可用或者繁忙的时候consumer会被自动切换到从slave读。

* 重试，死信
  **重试 Topic**：如果由于各种意外导致消息消费失败，那么该消息会自动被保存到重试Topic中，格式为“%RETRY%消费者组”，在订阅的时候会自动订阅这个重试Topic。
  进入重试队列的消息有16次重试机会，每次都会按照一定的时间间隔进行，只要正常消费或者重试消费中有一次消费成功，就算消费成功。
  ![image-20210404084012901](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210404084012901.png?lastModify=1617587790)
  **死信Topic**：死信Topic名字格式为“%DLQ%消费者组名”。如果正常消费1次失败，重试16次失败，那么消息会被保存到死信Topic中，进入死信Topic的消息不能被再次消费。RocketMQ认为，如果17次机会都失败了，说明生产者发送消息的格式发生了变化，或者消费服务出现了问题，需要人工介入处理。

* Rebalance
  Rebalance（重平衡）机制，用于在发生Broker掉线、Topic扩容和缩容、消费者扩容和缩容等变化时，自动感知并调整自身消费，以尽量减少甚至避免消息没有被消费。
  触发时机

  * Consumer启动时 启动之后会立马进行Rebalance
  * Consumer运行中 运行中会监听Broker发送过来的Rebalance消息，以及Consumer自身的定时任务（每隔20s）触发的Rebalance
  * Consumer停止运行 停止时没有直接的调用Rebalance，而是会通知Broker自己下线了，然后Broker会通知其余的Consumer进行Rebalance。

  Rebalance 核心逻辑主要在 client 侧。首先，Consumer 从 Broker 拉取自己订阅的所有 Topic 和 ConsumerGroup 下的消费者信息。然后会对 Topic 中 MessageQueue 和 消费者ID 进行排序，然后用消息队列默认分配算法来进行分配。排序是关键步骤，因为 consumer 之间并不通信，通过排序这样可以保证每一个 consumer 的视图一致，然后通过相同的分配算法，就能达到分配结果的一致。
  ![image-20210404094508003](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210404094508003.png?lastModify=1617587790)

#### 消费进度保存

集群消费模式中，消费进度保存在broker 上，而广播模式中保存在客户端本地。

消费进度是消费者组之间互相隔离的，Broker 上通过 Topic + 消费者组名称作为 key，value 中分别记录每个 MessageQueue 对应该消费者组的消费偏移量 offset。

消费者侧会记录自己的消费进度到内存中的 OffsetTable，然后定时提交到 Broker 侧。

由于一批消息的消费次序不确定，可能下标大的消息先被消费结束，下标小的由于延时尚未被消费，此时消费者向 Broker 提交的 offset 应该是已被消费的最小下标，从而保证消息不被遗漏，但缺点在于可能重复消费消息。

#### 消费方式

从用户应用的角度而言提供了两种消费形式：Pull、Push。

* Pull
  用户主动Pull消息，自主管理位点，可以灵活地掌控消费进度和消费速度，适合流计算、消费特别耗时等特殊的消费场景。缺点也显而易见，需要从代码层面精准地控制消费，对开发人员有一定要求。
* Push
  代码接入非常简单，适合大部分业务场景。缺点是灵活度差，在了解其消费原理后，排查消费问题方可简单快捷。Push 模式的实现是使用长轮询的方式，并非由 broker 推送。

![image-20210404100714552](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210404100714552.png?lastModify=1617587790)

#### 消息过滤

RocketMQ 的消费者可以根据 tag 进行消息过滤。消费者在 Pull 消息时，RocketMQ Broker 会根据 Tag 的 Hashcode 进行对比。如果不满足条件，消息不会返回给消费者，以节约宽带。字符串比较的速度相较Hashcode慢。Hashcode对比是数字比较，Java底层可以直接通过位运算进行对比，而字符串对比需要按照字符顺序比较，相比位运算更加耗时。由于Hashcode对比有Hash碰撞的危险，所以客户端会进行 二次 tag 过滤。

## Brocker

![img](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/rocketmq_design_1.png?lastModify=1617587790)

#### 复制和刷盘

复制是指Broker与Broker之间的数据同步方式，分为同步和异步两种。

* 同步复制时，生产者会等待同步复制成功后，才返回生产者消息发送成功。
* 异步复制时，消息写入Master Broker后即为写入成功，此时系统有较低的写入延迟和较大的系统吞吐量。

刷盘是指数据发送到Broker的内存（通常指PageCache）后，以何种方式持久化到磁盘。

* 同步刷盘时，生产者会等待数据持久化到磁盘后，才返回生产者消息发送成功，可靠性极强。
* 异步刷盘时，消息写入PageCache即为写入成功，到达一定量时自动触发刷盘。此时系统有非常低的写入延迟和非常大的系统吞吐量。

#### 存储

RocketMQ 主要存储的文件包括Comitlog文件、ConsumeQueue 文件、IndexFile文件。

* CommitLog：存储消息的元数据。RocketMQ将所有主题的消息存储在同一个文件中，确保消息发送时顺序写文件，尽最大能力确保消息发送的高性能与高吞吐量。每个文件大小一般是1GB，可以通过mapedFileSizeCommitLog进行配置。
* ConsumerQueue：存储消息在CommitLog的索引。由于消息中间件一般是基于消息主题的订阅机制，这样便给按照消息主题检索消息带来了极大的不便。为了提高消息消费的效率，RocketMQ引入了ConsumeQueue消息队列文件，每个消息主题包含多个消息消费队列，每一个消息队列有一个消息文件。每个消费队列其实是commitlog的一个索引，提供给消费者做拉取消息、更新位点使用。
  ![image-20210404214700195](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210404214700195.png?lastModify=1617587790)
* IndexFile：IndexFile索引文件，其主要设计理念就是为了加速消息的检索性能，根据消息的属性快速从Commitlog文件中检索消息，提供了一种通过key或者时间区间来查询消息的方法。全部的文件都是按照消息key创建的Hash索引。文件名是用创建时的时间戳命名的。

#### 高可用

主从同步

* 4.5之前：提供Master Broker 和 Slave Broker之间的数据同步功能，以及主从切换。 当master挂了, 写入服务就不可用了，但是  slave 服务仍然提供可读服务。
* 4.5之后：引入了Dleger，使用Raft共识算法, 在master故障后自动选举新leader。

## 其他特性

#### 消息顺序

消息有序指的是一类消息消费时，能按照发送的顺序来消费。

顺序消息分为全局顺序消息与分区顺序消息，全局顺序是指某个Topic下的所有消息都要保证顺序；部分顺序消息只要保证每一组消息被顺序消费即可。

* 全局顺序 对于指定的一个 Topic，所有消息按照严格的先入先出（FIFO）的顺序进行发布和消费。 适用场景：性能要求不高，所有的消息严格按照 FIFO 原则进行消息发布和消费的场景。
  * 全局消息的实现方式：一个全局顺序 topic 只创建一个Message Queue
* 分区顺序 对于指定的一个 Topic，所有消息根据 sharding key 进行区块分区。 同一个分区内的消息按照严格的 FIFO 顺序进行发布和消费。 Sharding key 是顺序消息中用来区分不同分区的关键字段，和普通消息的 Key 是完全不同的概念。 适用场景：性能要求高，以 sharding key 作为分区字段，在同一个区块中严格的按照 FIFO 原则进行消息发布和消费的场景。

#### 延时消息

定时消息（延迟队列）是指消息发送到broker后，不会立即被消费，等待特定时间投递给真正的topic。 broker有配置项messageDelayLevel，默认值为“1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h”，18个level。可以配置自定义messageDelayLevel。注意，messageDelayLevel是broker的属性，不属于某个topic。发消息时，设置delayLevel等级即可：msg.setDelayLevel(level)。level有以下三种情况：

* level == 0，消息为非延迟消息
* 1<=level<=maxLevel，消息延迟特定时间，例如level==1，延迟1s
* level > maxLevel，则level== maxLevel，例如level==20，延迟2h

定时消息会暂存在名为SCHEDULE_TOPIC_XXXX的topic中，并根据delayTimeLevel存入特定的queue，queueId = delayTimeLevel – 1，即一个queue只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。broker会调度地消费SCHEDULE_TOPIC_XXXX，将消息写入真实的topic。

需要注意的是，定时消息会在第一次写入和调度写入真实topic时都会计数，因此发送数量、tps都会变高。

#### 回溯消费

回溯消费是指Consumer已经消费成功的消息，由于业务上需求需要重新消费，要支持此功能，Broker在向Consumer投递成功消息后，消息仍然需要保留。

RocketMQ支持按照时间回溯消费，时间维度精确到毫秒。

#### 事务消息

RocketMQ事务消息（Transactional Message）是指应用本地事务和发送消息操作可以被定义到全局事务中，要么同时成功，要么同时失败。RocketMQ的事务消息提供类似 X/Open XA 的分布事务功能，通过事务消息能达到分布式事务的最终一致。

![img](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/1612267451228_fd9dadb0d4334489191a066f3f375cf7.png?lastModify=1617587790)

事务消息发送及提交：

(1) 发送消息（Half Message）。

(2) 服务端响应消息ACK写入结果。

(3) 根据发送结果执行本地事务（如果写入失败，此时half消息对业务不可见，本地逻辑不执行）。

(4) 根据本地事务状态执行Commit或者Rollback（Commit操作生成消息索引，消息对消费者可见）

补偿流程：

(1) 对没有Commit/Rollback的事务消息（pending状态的消息），从服务端发起一次“回查”

(2) Producer收到回查消息，检查回查消息对应的本地事务的状态

(3) 根据本地事务状态，重新Commit或者Rollback

其中，补偿阶段用于解决消息Commit或者Rollback发生超时或者失败的情况。

Half Message 并未真正进入Topic的queue，而是用了临时queue来放所谓的half message，等提交事务后才会真正的将half message转移到topic下的queue。

#### 至少一次

至少一次(At least Once)指每个消息必须投递一次。Consumer先Pull消息到本地，消费完成后，才向服务器返回ack，如果没有消费一定不会ack消息，但是可能会有 重复消费。

