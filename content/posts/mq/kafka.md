---
title: "Kafka 设计与理解"
date: 2021-05-02T11:22:38+08:00
draft: false
tags: ["Kafka"]
---

### 整体架构

![preview](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/v2-aa36f2bbc1a6ff0d8f03aad80759bb01_r.jpg)

> 图片来自 https://zhuanlan.zhihu.com/p/103249714

### Producer

负责发布消息到Kafka broker。Producer发送消息到broker时，会根据分区策略选择将其存储到哪一个Partition。

常规的有轮询，随机等策略，主要是为了将消息均衡的发送到各个 partition，提高并行度，从而提高吞吐。常用的还有按 key 哈希，主要是为了实现业务 partition 有序的需求。

### Consumer

消息消费者，从Kafka broker读取消息的客户端。

#### Consumer Group

Consumer Group是Kafka提供的可扩展且具有容错性的消费者机制，每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。

- Consumer Group下可以有一个或多个Consumer实例。这里的实例可以是一个单独的进程，也可以是同一进程下的线程。
- Group ID是一个字符串，在一个Kafka集群中，它标识唯一的一个Consumer Group。
- Consumer Group下所有实例订阅的主题的单个分区，只能分配给组内的某个Consumer实例消费。

#### 进度提交

消费者在消费的过程中需要记录自己消费了多少数据，即消费位置信息。在Kafka中，这个位置信息有个专门的术语：位移（Offset）。

#### Rebalance

何时触发 rebalance

1. 组成员数量发生变化，有新成员加入，或者有成员实例崩溃退出。
2. 订阅主题数量发生变化：Consumer Group可以使用正则表达式的方式订阅主题，比如果consumer.subscribe(Pattern.compile(“t.*c”))就表明该Group订阅所有以字母t开头、字母c结尾的主题。在Consumer Group的运行过程中，你新创建了一个满足这样条件的主题，那么该Group就会发生Rebalance
3. 订阅主题的分区数发生变化，如主题扩容。

Rebalance本质上是一种协议，规定了一个Consumer Group下的所有Consumer如何达成一致，来分配订阅Topic的每个分区Topic的每个分区。

比如某个Group下有20个Consumer实例，它订阅了一个具有100个分区的Topic。正常情况下，Kafka平均会为每个Consumer分配5个分区。这个分配的过程就叫Rebalance。

##### Rebalance 的流程

在消费者端，重平衡分为两个步骤：分别是 加入组 和 等待领导消费者（Leader Consumer）分配方案。这两个步骤分别对应两类特定的请求：JoinGroup请求和SyncGroup请求。

当组内成员加入组时，它会向 协调者 发送JoinGroup请求（后面会介绍协调者）。在该请求中，每个成员都要将自己订阅的主题上报，这样协调者就能收集到所有成员的订阅信息。一旦收集了全部成员的JoinGroup请求后，协调者会从这些成员中选择一个担任这个消费者组的领导者。

通常情况下，第一个发送JoinGroup请求的成员自动成为领导者。你一定要注意区分这里的领导者和之前我们介绍的领导者副本，它们不是一个概念。这里的领导者是具体的消费者实例，它既不是副本，也不是协调者。领导者消费者的任务是收集所有成员的订阅信息，然后根据这些信息，制定具体的分区消费分配方案。

选出领导者之后，协调者会把消费者组订阅信息封装进JoinGroup请求的响应体中，然后发给领导者，由领导者统一做出分配方案后，进入到下一步：发送SyncGroup请求。

在这一步中，领导者向协调者发送SyncGroup请求，将刚刚做出的分配方案发给协调者。值得注意的是，其他成员也会向协调者发送SyncGroup请求，只不过请求体中并没有实际的内容。这一步的主要目的是让协调者接收分配方案，然后统一以SyncGroup响应的方式分发给所有成员，这样组内所有成员就都知道自己该消费哪些分区了。



![20210429091039](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/20210429091039.png)



JoinGroup请求的主要作用是将组成员订阅信息发送给领导者消费者，待领导者制定好分配方案后，重平衡流程进入到SyncGroup请求阶段。



![image-20210429091619717](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210429091619717.png)

SyncGroup请求的主要目的，就是让协调者把领导者制定的分配方案下发给各个组内成员。当所有成员都成功接收到分配方案后，消费者组进入到Stable状态，即开始正常的消费工作。

正常情况下，每个组内成员都会定期汇报位移给协调者。当重平衡开启时，协调者会给予成员一段缓冲时间，要求每个成员必须在这段时间内快速地上报自己的位移信息，然后再开启正常的JoinGroup/SyncGroup请求发送。

### Broker

Kafka集群包含一个或多个服务器，这种服务器被称为broker。一个 broker 可以容纳多个 topic。brocker 是 kafka 中的核心组件，负责消息的存储，分区，路由信息的管理等。

#### **Partition**

Kafka中的分区机制指的是将每个主题划分成多个分区（Partition），每个分区是一组有序的消息日志。生产者生产的每条消息只会被发送到一个分区中，也就是说如果向一个双分区的主题发送一条消息，这条消息要么在分区0中，要么在分区1中。

如你所见，Kafka的分区编号是从0开始的，如果Topic有100个分区，那么它们的分区号就是从0到99。讲到这里，你可能有这样的疑问：刚才提到的副本如何与这里的分区联系在一起呢？实际上，副本是在分区这个层级定义的。

每个分区下可以配置若干个副本，其中只能有1个领导者副本和N-1个追随者副本。生产者向分区写入消息，每条消息在分区中的位置信息由一个叫位移（Offset）的数据来表征。分区位移总是从0开始，假设一个生产者向一个空分区写入了10条消息，那么这10条消息的位移依次是0、1、2、...、9。

Kafka Broker 使用消息日志（Log）来保存数据，一个日志就是磁盘上一个只能追加写（Append-only）消息的物理文件。因为只能追加写入，故避免了缓慢的随机I/O操作，改为性能较好的顺序I/O写操作，这也是实现Kafka高吞吐量特性的一个重要手段。

不停地向一个日志写入消息，最终也会耗尽所有的磁盘空间，因此Kafka必然要定期地删除消息以回收磁盘。简单来说就是通过日志段（Log Segment）机制。在Kafka底层，一个日志又近一步细分成多个日志段，消息被追加写到当前最新的日志段中，当写满了一个日志段后，Kafka会自动切分出一个新的日志段，并将老的日志段封存起来。Kafka在后台还有定时任务会定期地检查老的日志段是否能够被删除，从而实现回收磁盘空间的目的。

#### Topic

Topic 是一种逻辑上的分区，是同一类消息的集合，每一个消息只能属于一个 Topic。

为了使得Kafka的吞吐率提高，物理上把Topic分成一个或多个Partition，每个Partition在物理上对应一个文件夹，该文件夹下存储这个Partition的所有消息和索引文件。

每个日志文件都是一个`log entry`序列，每个`log entry`包含一个4字节整型数值（值为N+5），1个字节的”magic value”，4个字节的CRC校验码，其后跟N个字节的消息体。每条消息都有一个当前Partition下唯一的64字节的offset，它指明了这条消息的起始位置。磁盘上存储的消息格式如下：

```　　
message length ：4 bytes (value: 1+4+n)
“magic” value ： 1 byte
crc ： 					4 bytes
payload ： 			n bytes
```

这个`log entry`并非由一个文件构成，而是分成多个segment，每个segment以该segment第一条消息的offset命名并以“.kafka”为后缀。另外会有一个索引文件，它标明了每个segment下包含的`log entry`的offset范围。

![img](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/partition_segment.png)

> 图片来自：http://www.jasongj.com/2015/03/10/KafkaColumn1/

因为每条消息都被append到该Partition中，属于顺序写磁盘。但是如果 partition 过多，就会退化成随机写磁盘，因为 kafka 是每个 partition 一个目录去写，不同于 rokcetmq 的所有 queue 写一个文件。

#### Kafka 控制器

控制器组件（Controller），是Apache Kafka的核心组件。它的主要作用是在Apache ZooKeeper的帮助下管理和协调整个Kafka集群理和协调整个Kafka集群。集群中任意一台Broker都能充当控制器的角色，但是，在运行过程中，只能有一个Broker成为控制器，行使其管理和协调的职责。换句话说，每个正常运转的Kafka集群，在任意时刻都有且只有一个控制器

Broker在启动时，会尝试去ZooKeeper中创建/controller节点。Kafka当前选举控制器的规则是：第一个成功创建/controller节点的Broker会被指定为控制器第一个成功创建/controller节点的Broker会被指定为控制器。控制器是做什么的？控制器是做什么的？我们经常说，控制器是起协调作用的组件，那么，这里的协调作用到底是指什么呢？我想了一下，控制器的职责大致可以分为5种，我们一起来看看。

- 主题管理（创建、删除、增加分区）：控制器帮助我们完成对Kafka主题的创建、删除以及分区增加的操作。

- 分区重分配。

- Preferred领导者选举。

- 集群成员管理（新增Broker、Broker主动关闭、Broker宕机）：自动检测新增Broker、Broker主动关闭及被动宕机。

  这种自动检测是依赖于前面提到的Watch功能和ZooKeeper临时节点组合实现的。比如，控制器组件会利用Watch机制检 ZooKeeper的`/brokers/ids`节点下的子节点数量变更。目前，当有新Broker启动后，它会在`/brokers`下创建专属的znode节点。一旦创建完毕，ZooKeeper会通过Watch机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化，进而开启后续的新增Broker作业。侦测Broker存活性则是依赖于刚刚提到的另一个机制：临时节点。每个Broker启动后，会在`/brokers/ids`下创建一个临时znode。当Broker宕机或主动关闭后，该Broker与ZooKeeper的会话结束，这个znode会被自动删除。同理，ZooKeeper的Watch机制将这一变更推送给控制器，这样控制器就能知道有Broker关闭或宕机了，从而进行“善后”。

- 向其他Broker提供数据服务：控制器上保存了最全的集群元数据信息，其他所有Broker会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。

  这些数据其实在ZooKeeper中也保存了一份。每当控制器初始化时，它都会从ZooKeeper上读取对应的元数据并填充到自己的缓存中。有了这些数据，控制器就能对外提供数据服务了。这里的对外主要是指对其他Broker而言，控制器通过向这些Broker发送请求的方式将这些数据同步到其他Broker上。

  - 所有主题信息。包括具体的分区信息，比如领导者副本是谁，ISR集合中有哪些副本等。
  - 所有Broker信息。包括当前都有哪些运行中的Broker，哪些正在关闭中的Broker等。
  - 所有涉及运维任务的分区。包括当前正在进行Preferred领导者选举以及分区重分配的分区列表。

#### 控制器故障转移（Failover）

在Kafka集群运行过程中，只能有一台Broker充当控制器的角色，那么这就存在单点失效（Single Point of Failure）的风险。

当运行中的控制器突然宕机或意外终止时，Kafka能够快速地感知到（通过 ZK 的 watch 机制），并立即重新选出新的的控制器。这个过程就被称为Failover，该过程是自动完成的，无需你手动干预。

#### 高水位和LeaderEpoch

##### HW 和 LEO

- LEO（last end offset）：日志末端位移，记录了该副本对象底层日志文件中下一条消息的位移值，副本写入消息的时候，会自动更新 LEO 值。

- HW（high watermark）：HW 一定不会大于 LEO 值，小于 HW 值的消息被认为是“已提交，对消费者可见。



![image-20210502100333745](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210502100333745.png)

Log End Offset，简写是LEO，它表示副本写入下一条消息的位移值。同一个副本对象，其高水位值不会大于LEO值同一个副本对象。

Kafka所有副本都有对应的高水位和LEO值，而不仅仅是Leader副本。只不过Leader副本比较特殊，Kafka使用Leader副本的高水位来定义所在分区的高水位。

leader 会保存两个类型的 LEO 值，一个是自己的 LEO，另一个是 remote LEO 值，remote LEO 值就是 follower 副本的 LEO 值，意味着 follower 副本的 LEO 值会保存两份，一份保存到 leader 副本中，一份保存到自己这里。

**LEO 更新机制：**

1. leader 副本自身的 LEO 值更新：在 Producer 消息发送过来时，即 leader 副本当前最新存储的消息位移位置 +1；
2. follower 副本自身的 LEO 值更新：从 leader 副本中 fetch 到消息并写到本地日志文件时，即 follower 副本当前同步 leader 副本最新的消息位移位置 +1；
3. leader 副本中的 remote LEO 值更新：每次 follower 副本发送 fetch 请求都会包含 follower 当前 LEO 值，leader 拿到该值就会尝试更新 remote LEO 值。

**leader HW 更新机制：**

leader HW 更新分为故障时更新与正常时更新：

故障时更新：

1. 副本被选为 leader 副本时：当某个 follower 副本被选为分区的 leader 副本时，kafka 就会尝试更新 HW 值；
2. 副本被踢出 ISR 时：如果某个副本追不上 leader 副本进度，或者所在 broker 崩溃了，导致被踢出 ISR，leader 也会检查 HW 值是否需要更新，毕竟 HW 值更新只跟处于 ISR 的副本 LEO 有关系。

正常时更新：

1. producer 向 leader 副本写入消息时：在消息写入时会更新 leader LEO 值，因此需要再检查是否需要更新 HW 值；
2. leader 处理 follower FETCH 请求时：follower 的 fetch 请求会携带 LEO 值，leader 会根据这个值更新对应的 remote LEO 值，同时也需要检查是否需要更新 HW 值。

**follower HW 更新机制：**

1. follower 更新 HW 发生在其更新 LEO 之后，每次 follower Fetch 响应体都会包含 leader 的 HW 值，然后比较当前 LEO 值，取最小的作为新的 HW 值。



![img](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/20191030190621.png)



上图 stop4 说明 Follower副本的HW更新需要一轮额外的拉取请求才能实现。

如果把上面那个例子扩展到多个Follower副本，情况可能更糟，也许需要多轮拉取请求。也就是说，Leader副本高水位更新和Follower副本高水位更新在时间上是存在错配的。这种错配是很多“数据丢失”或“数据不一致”问题的根源。

###### 数据丢失举例

![img](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/kafka_leader_epoch_error_1.jpg)

第1步中的状态:

1. 副本B作为leader收到producer的m2消息并写入本地文件，等待副本A拉取。
2. 副本A发起消息拉取请求，请求中携带自己的最新的日志offset（LEO=1），B收到后更新自己的HW为1，并将HW=1的信息以及消息m2返回给A。
3. A收到拉取结果后更新本地的HW为1，并将m2写入本地文件。发起新一轮拉取请求（LEO=2），B收到A拉取请求后更新自己的HW为2，没有新数据只将HW=2的信息返回给A，并且回复给producer写入成功。此处的状态就是图中第一步的状态。

此时，如果没有异常，A会收到B的回复，得知目前的HW为2，然后更新自身的HW为2。

但在第2步A重启了，没有来得及收到B的回复，此时B仍然是leader。

**A重启之后会以自己的 HW 为标准截断自己的日志，因为A作为follower不知道多出的日志是否是被提交过的，防止数据不一致从而截断多余的数据并尝试从leader那里重新同步**

第3步，B崩溃了，min.isr设置的是1，所以zookeeper会从ISRs中再选择一个作为leader，也就是A，但是A的数据不是完整的，从而出现了数据丢失现象。

##### LeaderEpoch

Leader Epoch，我们大致可以认为是Leader版本。它由两部分数据组成。

- Epoch。一个单调增加的版本号。每当副本领导权发生变更时，都会增加该版本号。小版本号的Leader被认为是过期Leader，不能再行使Leader权力。

- 起始位移（Start Offset）。Leader副本在该Epoch值上写入的首条消息的位移。

  

> 举个例子来说明一下Leader Epoch。
>
> 假设现在有两个Leader Epoch<0, 0>和<1, 120>，那么，第一个Leader Epoch表示版本号是0，这个版本的Leader从位移0开始保存消息，一共保存了120条消息。之后，Leader发生了变更，版本号增加到1，新版本的起始位移是120。

Kafka Broker会在内存中为每个分区都缓存Leader Epoch数据，同时它还会定期地将这些信息持久化到一个checkpoint文件中。当Leader副本写入消息到磁盘时，Broker会尝试更新这部分缓存。如果该Leader是首次写入消息，那么Broker会向缓存中增加一个Leader Epoch条目，否则就不做更新。这样，每次有Leader变更时，新的Leader副本会查询这部分缓存，取出对应的Leader Epoch的起始位移，以避免数据丢失和不一致的情况



![img](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/kafka_leader_epoch_solve_1.jpg)

引用Leader Epoch机制后，Follower副本B重启回来后，需要向A发送一个特殊的请求去获取Leader的LEO值。

在这个例子中，该值为2。当获知到Leader LEO=2后，B发现该LEO值不比它自己的LEO值小，而且缓存中也没有保存任何起始位移值 > 2的Epoch条目，因此B无需执行任何日志截断操作。

Leader Epoch 是对高水位机制的一个明显改进，即副本是否执行日志截断不再依赖于高水位进行判断。

#### 高可用

#### Replication

副本（Replica）本质是一个只能追加写消息的提交日志。根据Kafka副本机制的定义，同一个分区下的所有副本保存有相同的消息序列，这些副本分散保存在不同的Broker上。

Kafka定义了两类副本：领导者副本(Leader)和追随者副本(Follower)。一主多从，这些副本保存着相同的数据。领导者副本对外提供服务，追随者副本只是不断异步拉取领导者副本的消息，不对外提供服务。

当领导者副本挂掉了，或者说领导者副本所在的Broker宕机时，Kafka依托于ZooKeeper提供的监控功能能够实时感知到，并立即开启新一轮的领导者选举，从追随者副本中选一个作为新的领导者。老Leader副本重启回来后，只能作为追随者副本加入到集群中。

##### 为什么kafka的追随者副本不对外服务

- 方便实现“Read-your-writes”。所谓Read-your-writes，顾名思义就是，当你使用生产者API向Kafka成功写入消息后，马上使用消费者API去读取刚才生产的消息。
- 方便实现单调读（Monotonic Reads）。什么是单调读呢？就是对于一个消费者用户而言，在多次消费消息时，它不会看到某条消息一会儿存在一会儿不存在。

##### ISR

Kafka既不是完全的同步复制，也不是完全的异步复制，而是基于ISR的动态复制方案。

ISR 可以把他看成主从结构的升级版。对于每一个 partiton，可以分配一个或多个 Broker。 其中一个作为主节点，剩余的作为跟随者，跟随者会保存一个 partition 副本。生产者将消息发送到主节点后，主节点会广播给所有跟随者，跟随者收到后返回确认信息给主节点。 用户可以自由的配置副本数及当有几个副本写成功后，则认为消息成功保存。且同时，会在 ZooKeeper 上维护一个可用跟随者列表，列表中记录所有数据和主节点完全同步的跟随者列表。当主节点发生故障时，在列表中选择一个跟随者作为新的主节点提供服务。在这种策略下，假设总共有 m 个副本，要求至少有 n 个（0<n<m+1）副本写成功，则系统可以在最多 m-n 个机器故障的情况下保证可用性。

> 还有一种实现是基于 Raft 算法实现的多副本机制，具体细节可以参考官方的 paper。Raft 集群一般由奇数节点构成，如果要保证集群在 n 个节点故障的情况下可用，则至少需要有 2n+1 个节点。 与 ISR 方式相比，Raft 需要耗费更多的资源，但是整个复制和选举过程都是集群中的节点自主完成，不需要依赖 ZooKeeper 等第三者。 理论上 Raft 集群规模可以无限扩展而 ISR 模式下集群规模会受限于 ZooKeeper 集群的处理能力。 

ISR，也即In-sync Replica。每个Partition的Leader都会维护这样一个列表，该列表中，包含了所有与之同步的Replica（包含Leader自己）。每次数据写入时，只有ISR中的所有Replica都复制完，Leader才会将其置为Commit，它才能被Consumer所消费。

Broker端参数 `replica.lag.time.max.ms`，这个参数的含义是Follower副本能够落后Leader副本的最长时间间隔，当前默认值是10秒。这就是说，只要一个Follower副本落后Leader副本的时间不连续超过10秒，那么Kafka就认为该Follower副本与Leader是同步的，即使此时Follower副本中保存的消息明显少于Leader副本中的消息。我们在前面说过，Follower副本唯一的工作就是不断地从Leader副本拉取消息，然后写入到自己的提交日志中。如果这个同步过程的速度持续慢于Leader副本的消息写入速度，那么在replica.lag.time.max.ms时间后，此Follower副本就会被认为是与Leader副本不同步的，因此不能再放入ISR中。此时，Kafka会自动收缩ISR集合，将该副本“踢出”ISR。值得注意的是，倘若该副本后面慢慢地追上了Leader的进度，那么它是能够重新被加回ISR的。这也表明，ISR是一个动态调整的集合，而非静态不变的。

###### Unclean领导者选举（Unclean Leader Election）

既然ISR是可以动态调整的，那么自然就可以出现这样的情形：ISR为空。因为Leader副本天然就在ISR中，如果ISR为空了，就说明Leader副本也“挂掉”了，Kafka需要重新选举一个新的Leader。

此时该怎么选举新Leader呢？Kafka把所有不在ISR中的存活副本都称为非同步副本Kafka把所有不在ISR中的存活副本都称为非同步副本。通常来说，非同步副本落后Leader太多，因此，如果选择这些副本作为新Leader，就可能出现数据的丢失。毕竟，这些副本中保存的消息远远落后于老Leader中的消息。在Kafka中，选举这种副本的过程称为Unclean领导者选举。Broker端参数Broker端参数unclean.leader.election.enable控制是否允许Unclean领导者选举unclean.leader.election.enable控制是否允许Unclean领导者选举。

开启Unclean领导者选举可能会造成数据丢失，但好处是，它使得分区Leader副本一直存在，不至于停止对外提供服务，因此提升了高可用性。反之，禁止Unclean领导者选举的好处在于维护了数据的一致性，避免了消息丢失，但牺牲了高可用性。



>  推荐阅读
>
> [为什么Kafka需要Leader Epoch](https://t1mek1ller.github.io/2020/02/15/kafka-leader-epoch/)
>
> [Kafka水位(high watermark)与leader epoch的讨论](https://www.cnblogs.com/huxi2b/p/7453543.html)
>
> [Kafka高性能架构之道](http://www.jasongj.com/kafka/high_throughput/)