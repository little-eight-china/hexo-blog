---
title: 'kafka的一万字精华总结'
date: 2021-09-27 22:50:20
categories: 
  - mq
tags: 
  - little_eight
---
## 什么是 Kafka
&ensp;&ensp;&ensp;&ensp;Kafka 是一个分布式的，支持多分区、多副本的分布式消息流平台，它同时也是一款开源的**基于发布订阅模式的消息引擎系统，**简单来说就是一个分布式消息队列。
&ensp;&ensp;&ensp;&ensp;在2.8版本之前，无论是Kafka 集群，还是consumer都依赖于zookeeper集群保存一些meta信息，来保证系统可用性。2.8版本之后使用内嵌的KRaft作为zookeeper部分功能的替代品。
![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/1.png)
<!--more-->
&ensp;&ensp;&ensp;&ensp;如上图所示，一个典型的 Kafka 集群中包含若干Producer，若干broker（Kafka支持水平扩展，一般broker数量越多，集群吞吐率越高），若干Consumer Group，以及一个Zookeeper集群。Kafka通过Zookeeper管理集群配置，选举leader，以及在Consumer Group发生变化时进行rebalance。Producer使用push模式将消息发布到broker，Consumer使用pull模式从broker订阅并消费消息。
​

## 使用场景
### 活动跟踪
&ensp;&ensp;&ensp;&ensp;kafka可以用来跟踪用户行为，比如我们经常回去淘宝购物，你打开淘宝的那一刻，你的登陆信息，登陆次数都会作为消息传输到 kafka，当你浏览购物的时候，你的浏览信息，你的搜索指数，你的购物爱好都会作为一个个消息传递给 kafka，这样就可以生成报告，可以做智能推荐，购买喜好等。
### 传递消息
&ensp;&ensp;&ensp;&ensp;kafka另外一个基本用途是传递消息，应用程序向用户发送通知就是通过传递消息来实现的，这些应用组件可以生成消息，而不需要关心消息的格式，也不需要关心消息是如何发送的；度量指标：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。
### 日志记录
&ensp;&ensp;&ensp;&ensp;Kafka 的基本概念来源于提交日志，比如我们可以把数据库的更新发送到 kafka上，用来记录数据库的更新时间，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等。
### 流式处理
&ensp;&ensp;&ensp;&ensp;流式处理是有一个能够提供多种应用程序的领域；限流削峰：kafka多用于互联网领域某一时刻请求特别多的情况下，可以把请求写入kafka中，避免直接请求后端程序导致服务崩溃。
​

## 基本数据结构
### Producer
&ensp;&ensp;&ensp;&ensp;生产者，发送消息的一方，负责创建消息发给kafka，会把消息发送到指定的topic里。
### Comsumer
&ensp;&ensp;&ensp;&ensp;消费者，连接kafka并接收消息，对指定的topic进行消费消息。
### Comsumer group
&ensp;&ensp;&ensp;&ensp;一个消费者组可以包含n个消费者，同一组的消费者会去消费指定的topic，保证不会重复消费同一条消息。
&ensp;&ensp;&ensp;&ensp;因为topic里包含多个partition，kafka规定一个partition不能由多个comsumer消费，如果消费组的comsumer数量小于partition的，就会导致多余的comsumer线程空闲，如果大于那么comsumer会进行轮询直到消费完所有的partition，所以**消费组里的comsumer的数量跟partition一致可达到最高利用率**。
​

### Broker
&ensp;&ensp;&ensp;&ensp;服务代理节点，也就是服务节点，kafka的服务器。
### Topic
&ensp;&ensp;&ensp;&ensp;kafka的消息以topic为单位进行划分，生产者将消息发送到特定的topic，而消费者负责订阅topic的消息并进行消费。
### Partition
​

&ensp;&ensp;&ensp;&ensp;topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。消息到来时会使用轮询制往每个partition写入，如下图。
![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/2.png)

&ensp;&ensp;&ensp;&ensp;kafka默认使用的是hash进行分区，所以同一个topic下的不同partition包含的消息是不同的。
&ensp;&ensp;&ensp;&ensp;物理上**每个partition对应的是一个文件夹**，其命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。而文件夹里存储的当然是文件，这个文件就是segment。
​

### Segment
&ensp;&ensp;&ensp;&ensp;partition包含多个segment，每个segment对应一个文件，partition全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。
&ensp;&ensp;&ensp;&ensp;segment可以手动指定大小（**log.segment.bytes**），当segment达到阈值时，将不再写数据。
&ensp;&ensp;&ensp;&ensp;记录只会被append到segment中，不会被单独删除或者修改，每个segment中的消息数量不一定相等。
&ensp;&ensp;&ensp;&ensp;而segment又分为索引跟手数据文件，2个文件是相关对应的，如下图：
![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/3.png)



- **索引文件**：以.index结尾的文件，采取**稀疏索引存储方式**，它减少索引文件大小。其中的数据指向对应数据文件的消息的物理偏移地址。在查找的时候利用索引文件快速定位到数据文件的位置实现快速查找，如下图。
- **数据文件**：存储消息的文件，由多个messager组成。

![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/4.png)


### Messager
&ensp;&ensp;&ensp;&ensp;消息的本体。其结构为：

- **offset**：偏移量，8byte大小。在parition内的每条消息都有一个有序的id号，这个id号被称为偏移量,它可以唯一确定每条消息在parition内的位置。
- message size：消息大小，4byte大小。
- crc32：用crc32校验消息，4byte大小。
- magic： 表示本次发布Kafka服务程序协议版本号，1byte大小。
- attributes：表示为独立版本、或标识压缩类型、或编码类型，1byte大小。
- key length：表示key的长度,当key为-1时，K byte key字段不填。
- K byte key：设置消息的key，相同key的消息在不同时刻有不同的值，则只允许存在最新的一条消息。
- **value bytes payload**：实际数据。
### Offset
&ensp;&ensp;&ensp;&ensp;每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用于partition唯一标识一条消息。
​

### Replication
&ensp;&ensp;&ensp;&ensp;副本**，**kafka保证高可用的方式，同一个partition的数据可以在多个broker上存在多个副本，通常只有主副本对外提供读写服务，当主副本挂了之后，kafka会重新进行选举产生主副本。
​

## kafka的快
​

### 写的快
&ensp;&ensp;&ensp;&ensp;为了优化写入速度，kafka采用了两种技术，一种是顺序写入，一种是MMAP。
#### 顺序写入
&ensp;&ensp;&ensp;&ensp;因为硬盘是机械结构，每次读写都会寻址->写入，其中寻址是一个“机械动作”，它是最耗时的。所以硬盘最讨厌随机I/O，最喜欢顺序I/O。为了提高读写硬盘的速度，kafka就是使用顺序I/O。
​

#### Memory Mapped Files
&ensp;&ensp;&ensp;&ensp;即便是顺序写入磁盘，磁盘的访问速度还是不可能追上内存的。所以kafka的数据并不是实时的写入硬盘，它充分利用了现代操作系统的分页存储来利用内存，以此来提高I/O效率。Memory Mapped Files（后面简称MMAP）也被翻译成**内存映射文件**，在64位操作系统中一般可以表示20G的数据文件。它的工作原理是直接利用操作系统的Page来实现文件到物理内存的直接映射。完成映射之后，你对物理内存的操作会被同步到硬盘上（操作系统在适当的时候）。
&ensp;&ensp;&ensp;&ensp;通过MMAP，进程就可以像读写硬盘一样读写内存（当然是虚拟机内存），也不必关系内存的大小，因为有虚拟内存为我们兜底。使用这种方式可以获取很大的I/O提升，省去了用户空间到内核空间复制的开销（调用文件的read会有把数据先放到内核空间的内存中，然后再复制到用户空间的内存中）。
&ensp;&ensp;&ensp;&ensp;但是这样也有一个很明显的缺陷：不可靠，因为写到MMAP中的数据并没有被真正地写入到硬盘中，操作系统会在程序主动调用flush命令的时候才会把数据真正地写入到硬盘中。
&ensp;&ensp;&ensp;&ensp;MMAP其实是Linux中的一个函数，就是用来实现内存映射的。Java的NIO提供了一个MappedByteBuffer类来实现内存映射（因此Kafka是沾了Java的光，而不是Scala）。
​

### 读的快
&ensp;&ensp;&ensp;&ensp;为了优化读取速度，kafka采用了两种技术，一种是零拷贝，一种是批量压缩。
​

#### 零拷贝
&ensp;&ensp;&ensp;&ensp;传统的文件读写或者网络传输，通常需要将数据从**内核态**转换为**用户态**。应用程序读取用户态内存数据，写入文件 / Socket之前，需要从用户态转换为内核态之后才可以写入文件或者网卡当中。数据首先从磁盘读取到内核缓冲区，这里面的内核缓冲区就是页缓存（PageCache）。然后从内核缓冲区中复制到应用程序缓冲区（用户态），输出到输出设备时，又会将用户态数据转换为内核态数据，这样的话就相当于数据需要复制4次才能真正获取到。
![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/5.png)


&ensp;&ensp;&ensp;&ensp;在介绍零拷贝之前，我们先来看一个技术名词DMA（Direct Memory Access 直接内存访问）。它是现代电脑的重要特征之一，允许不同速度的硬件之间直接交互，而不需要占用CPU的中断负载。DMA传输将一个地址空间复制到另一个地址空间，当CPU 初始化这个传输之后，实际的数据传输是有DMA设备之间完成，这样可以大大的减少CPU的消耗。我们常见的硬件设备都支持DMA，如下图所示：
​

![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/6.png)

&ensp;&ensp;&ensp;&ensp;零拷贝技术经过 DMA（Direct Memory Access）技术将文件内容复制到内核态下的页缓存中。不过没有数据被复制到 socket缓冲区，相反只有包含数据的位置和长度的信息的文件描述符被加到 socket缓冲区中。DMA 引擎直接将数据从内核态中传递到网卡设备（协议引擎）。这里数据只经历了2次复制就从磁盘中传送出去了，而且上下文切换也变成了2次。
&ensp;&ensp;&ensp;&ensp;零拷贝是针对内核态而言的，数据在内核态下实现了零拷贝。
&ensp;&ensp;&ensp;&ensp;kafka基于sendfile实现了零拷贝，运行流程如下：

1. sendfile系统调用，文件数据被copy至内核缓冲区。
1. 再从内核缓冲区copy至内核中socket缓冲区。
1. 最后再socket缓冲区copy到协议引擎。
#### 批量压缩
&ensp;&ensp;&ensp;&ensp;在很多情况下，系统的瓶颈不是CPU或磁盘，而是网络IO，对于需要在广域网上的数据中心之间发送消息的数据流水线尤其如此。进行数据压缩会消耗少量的CPU资源,不过对于kafka而言,网络IO更应该需要考虑。

- 如果每个消息都压缩，会导致压缩率相对很低，所以Kafka使用了批量压缩，即将多个消息一起压缩而不是单个消息压缩。
- Kafka允许使用递归的消息集合，批量的消息可以通过压缩的形式传输并且在日志中也可以保持压缩格式，直到被消费者解压缩。
- Kafka支持多种压缩协议，包括Gzip和Snappy压缩协议。

​

## kafka日志清除策略
&ensp;&ensp;&ensp;&ensp;众所周知，物理空间并不是无限大的，所以所有程序都会有一定的清除无用数据的策略，来保证程序的正常运转。因为kafka存储的是以文件类型存储数据，所以并不像rabbitmq一样当消费者确认消费完成后就会删除该消息，而是采用2种策略来对数据日志文件进行清除。**log.cleanup.policy**可指定清理策略，分别为**delete**与**compact**。注意的是这2种策略是可以同时设置同时生效的。
### delete
&ensp;&ensp;&ensp;&ensp;会按照一定的属性来对数据日志文件进行清除。
&ensp;&ensp;&ensp;&ensp;kafka中所有用户创建的topic，默认均为此策略。

- log.retention.hours：指定数据保留的时间（默认为一周，168）
- log.retention.bytes：每个partition中保存的最大数据量大小（默认为-1，也就是无限大）
- log.retention.check.interval.ms： 指定间隔时间检查文件是否符合清除阈值，如果符合将其标记为deleted，也就是文件名后缀加.delete
- log.cleaner.delete.retention.ms: deleted文件保留多长时间，默认是1天
- log.cleaner.backoff.ms： 没有日志要清除时的休眠时间，默认15秒
### compact
&ensp;&ensp;&ensp;&ensp;会按照一定的属性来对消息进行清除，topic __consumer_offsets 默认为此策略，有以下特点：

- 根据messages中的key，进行删除操作
- 在active segment 被commit 后，会删除掉old duplicate keys
- 无限制的时间与空间的日志保留

​

&ensp;&ensp;&ensp;&ensp;注意的是清除message时是不会对剩下的进行重新排序，如果message已被删除当消费者凭借offset进行消费时会跳过不存在的message。
​

&ensp;&ensp;&ensp;&ensp;其中相关的属性为：

- segment.ms：在关闭一个active segment前，所需等待的最长时间。默认为7天
- segment.bytes：一个segment的最大大小，默认为1G。
- min.compaction .lag.ms：在一个message可以被compact前，所需等待的时间，默认为0。
- delete.retention.ms：在一条message被加上删除标记后，在实际删除前等待的时间，默认为24小时。
- min.Cleanable.dirty.ratio：若是设置的更高，则会有更高效的清理，但是更少的清理操作触发。若是设置的更低，则清理的效率稍低，但是会有更多的清理操作被触发，默认为0.5。

​

## kafka的消息可靠性
&ensp;&ensp;&ensp;&ensp;可靠性是有三方共同保证的。
### 生产者保证消息可靠性
&ensp;&ensp;&ensp;&ensp;生产者先把message发送到partition leader，再由leader发送给其他partition follower。而kafka为生产者提供了2种异步方式，一种是有回调函数的方法send(message)，一种是没有回调函数的send(message,callback)。

- **send(message)**

&ensp;&ensp;&ensp;&ensp;该方法可以将一条消息发送出去，但是对发送出去的消息没有掌控能力，无法得知其最后是不是到达了Kafka，所以这是一种不可靠的发送方式，但是也因为客户端只需要负责发送，所以具有较好的性能。

- **send(message,callback)**

&ensp;&ensp;&ensp;&ensp;该方法可以将一条消息发送出去，并且可以从callback回调中得到该条消息的发送结果，并且callback是异步回调，所以在兼具性能的情况下，也对消息具有比较好的掌控。而kafka里有一个ack的属性，这个可以保证生产者的消息安全送达kafka。其中：

- acks=0
acks = 0如果设置为零，那么生产者将完全不会管服务器是否收到消息。该记录将立即添加到套接字缓冲区中并视为已发送。并且重试配置不会生效（因为客户端通常不会知道任何故障）。返回值的偏移量将始终等于 -1。该方式具有最大的吞吐量，一般建议直接配合 send(msg)使用。
- acks=1
当leader接受到消息就会直接给客户端返回成功，一般情况下这种模式都能很好的保证数据的不丢失，只有在laeder接受到数据，然后还没来得及同步到follower，就挂掉了才会导致数据的丢失，这种概率还是比较小的。这也是默认的选择方式，兼具较好的吞吐和较高的可靠性。
- acks=all 或者 acks=-1
当leader接受到消息，并同步到了一定数量的follower，才向生产者发生成功的消息，同步到的follower数量由 broker 端的 min.insync.replicas 决定。除非一些不可抗力因素，这种方式基本可以确保数据的完全不丢失。

​

&ensp;&ensp;&ensp;&ensp;producer还有一个重试的机制，当发送消息失败时就会间隔指定时间再次发送。当重试次数超过预设的阈值后，需要怎么处理是由**你自己的业务场景决定**。常见的做法包括：

- 继续重试直到天荒地老，如果是网线断了这种方法非常低效。
- 如果业务对丢失消息不敏感，可以直接抛弃这条信息。
- 如果业务对丢消息敏感，可以把错误记录下来，然后告警让人工介入处理。

​

### broker保证消息可靠性
&ensp;&ensp;&ensp;&ensp;既然要保证消息可靠性，在集群中是避免不了主从选举的。
#### broker集群选举
&ensp;&ensp;&ensp;&ensp;在一个kafka集群中，有多个broker节点，但是它们之间需要选举出一个leader，其他的broker充当follower角色。集群中第一个启动的broker会通过在zookeeper中创建临时节点/controller来让自己成为控制器，其他broker启动时也会在zookeeper中创建临时节点，但是发现节点已经存在，所以它们会收到一个异常，意识到控制器已经存在，那么就会在zookeeper中创建watch对象，便于它们收到控制器变更的通知。
&ensp;&ensp;&ensp;&ensp;那么如果控制器由于网络原因与zookeeper断开连接或者异常退出，那么其他broker通过watch收到控制器变更的通知，就会去尝试创建临时节点/controller，如果有一个broker创建成功，那么其他broker就会收到创建异常通知，也就意味着集群中已经有了控制器，其他broker只需创建watch对象即可。
&ensp;&ensp;&ensp;&ensp;如果集群中有一个broker发生异常退出了，那么**控制器就会检查这个broker是否有分区的副本leader，如果有那么这个分区就需要一个新的leader，此时控制器就会去遍历其他副本，决定哪一个成为新的leader，同时更新分区的ISR集合**。
&ensp;&ensp;&ensp;&ensp;如果有一个broker加入集群中，那么控制器就会通过broker ID去判断新加入的broker中是否含有现有分区的副本，如果有，就会从分区副本中去同步数据。
&ensp;&ensp;&ensp;&ensp;集群中每选举一次控制器，就会通过zookeeper创建一个controller epoch，每一个选举都会创建一个更大，包含最新信息的epoch，如果有broker收到比这个epoch旧的数据，就会忽略它们，kafka也通过这个epoch来防止集群产生“脑裂”。
​

#### replication副本选举
​
&ensp;&ensp;&ensp;&ensp;kafka给partition设置了replication副本的概念，把这些副本均匀的分布到多个broker上，就保证了数据的安全，不再担心某个broker宕机后使其中的partition失效，如下图：
![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/7.png)


​

&ensp;&ensp;&ensp;&ensp;总共有三种副本角色：

- leader副本：也就是leader主副本，每个分区都有一个leader副本，为了保证数据一致性，所有的生产者与消费者的请求都会经过该副本来处理。
- follower副本：除了首领副本外的其他所有副本都是follower副本，follower副本不处理来自客户端的任何请求，只负责从leader副本同步数据，保证与首领保持一致。如果leader副本发生崩溃，就会从这其中选举出一个leader。
- 优先副本：创建分区时指定的优先leader。如果不指定，则为分区的第一个副本。

​

&ensp;&ensp;&ensp;&ensp;leader维护了一个动态的in-sync replica set (**ISR**)，意为和leader保持同步的follower集合，其中**min.insync.replicas**(默认为1)为ISR最小副本数，如果ISR中副本数小于该值则在接收生产者消息时就会以发送失败为结果。当ISR中的follower完成数据的同步之后，leader就会给producer发送ack。如果follower长时间未向leader同步数据，则该follower将被踢出ISR，该时间阈值由**replica.lag.time.max.ms**（默认30秒）参数设定。Leader发生故障之后，就会从ISR中选举新的leader。被踢出ISR的follower重连后依然会从leader中拉取数据，直到拉取的数据到达ISR的中partition的HW位置时，此被踢出的follower会重新加入ISR中。

&ensp;&ensp;&ensp;&ensp;在**副本所在的broker发生宕机后**，怎么处理呢？
&ensp;&ensp;&ensp;&ensp;首先先了解下三个名词：

- HW（High Watermark），俗称“高水位”，它表示了一个特定消息的偏移量（offset），消费只能拉取到这个offset之前的消息。
- LW（Low Watermark），俗称“低水位”，代表AR集合中最小的logStartOffset值，副本的拉取请求和删除请求都可能促使LW的增长。
- LEO（Log End Offset），它表示当前日志文件中下一条待写入消息的offset，LEO的大小相当于当前日志分区中最后一条消息的offset值+1。分区ISR集合中每个副本都会维护自身的LEO，而ISR集合中最小的LEO即为分区的HW，对于消费者而言，只能消费HW之前的消息。

![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/8.png)
​

​


- follower故障

&ensp;&ensp;&ensp;&ensp;follower发生故障后会被临时踢出ISR，待该follower恢复后，follower会读取本地磁盘记录的上次的HW，并将log文件高于HW的部分截取掉，从HW开始向leader进行同步。等该follower的LEO大于等于该Partition的HW，即follower追上leader之后，就可以重新加入ISR了。

- leader故障

&ensp;&ensp;&ensp;&ensp;leader发生故障之后，会从ISR中选出一个新的leader，之后，为保证多个副本之间的数据一致性，其余的follower会先将各自的log文件高于HW的部分截掉，然后从新的leader同步数据。**所以这只能保证副本之间的一致性，并不能保证数据不丢失或者不重复。**


### 消费者保证消息可靠性
&ensp;&ensp;&ensp;&ensp;在消费者对指定消息分区进行消息消费的过程中，会根据选用的api不同来保存消费的offset。

- 如果是kafka.javaapi.consumer.ConsumerConnector，我们会配置参数**zookeeper.connect**来消费。这种情况下，消费者的offset会更新到zookeeper的。consumers/{group}/offsets/{topic}/{partition}目录下。
- 如果是根据kafka默认的api来消费，即org.apache.kafka.clients.consumer.KafkaConsumer，我们会配置参数**bootstrap.servers**来消费。而其消费者的offset会更新到一个kafka自带的**__consumer_offsets**下面，查看当前group的消费进度，则要依靠kafka自带的工具kafka-consumer-offset-checker。

​

&ensp;&ensp;&ensp;&ensp;offset更新的方式，不区分是用的哪种api，大致分为两类：

- 自动提交，设置enable.auto.commit=true，更新的频率根据参数[**auto.commit.interval.ms**](http://auto.commit.interval.ms/)来定。这种方式也被称为at most once，fetch到消息后就可以更新offset，无论是否消费成功。
- 手动提交，设置enable.auto.commit=false，这种方式称为at least once。fetch到消息后，等消费完成再调用方法**consumer.commitSync()**，手动更新offset；如果消费失败，则offset也不会更新，此条消息会被重复消费一次。设置手动提交的话，就可保证消息消费到。

​

## 消息积压
&ensp;&ensp;&ensp;&ensp;通常情况下，企业中会采取轮询或者随机的方式，通过kafka的producer向kafka集群生产数据，来尽可能保证kafka分区之间的数据是均匀分布的。
&ensp;&ensp;&ensp;&ensp;在分区数据均匀分布的前提下，如果我们针对要处理的topic数据量等因素，设计出合理的kafka分区数量。对于一些实时任务，比如Flink和Kafka集成的应用，消费端不存在长时间"挂掉"的情况即数据一直在持续被消费，那么一般不会产生kafka数据积压的情况。
&ensp;&ensp;&ensp;&ensp;但是这些都是有前提的，当一些意外或者不合理的分区数设置情况的发生，积压问题就不可避免。
### 典型案例
**实时/消费任务挂掉了**
&ensp;&ensp;&ensp;&ensp;比如，我们写的实时应用因为某种原因挂掉了，并且这个任务没有被监控程序监控发现通知相关负责人，负责人又没有写自动拉起任务的脚本进行重启。
&ensp;&ensp;&ensp;&ensp;那么在我们重新启动这个实时应用进行消费之前，这段时间的消息就会被滞后处理，如果数据量很大，可就不是简单重启应用直接消费就能解决的。
#### 消费组不断重平衡
&ensp;&ensp;&ensp;&ensp;消费组重平衡期间是无法消费的，当我们的消费程序出现间接性消费缓慢或者超时异常的时候，可能是遇到消费者组重平衡了。
**kafka分区数设置的不合理（太少）和消费者"消费能力"不足**
&ensp;&ensp;&ensp;&ensp;kafka单分区生产消息的速度qps通常很高，如果消费者因为某些原因（比如受业务逻辑复杂度影响，消费时间会有所不同），就会出现消费滞后的情况。
&ensp;&ensp;&ensp;&ensp;此外，kafka分区数是kafka并行度调优的最小单元，如果kafka分区数设置的太少，会影响consumer消费的吞吐量。
**kafka消息的key不均匀，导致分区间数据不均衡**
&ensp;&ensp;&ensp;&ensp;在使用kafka producer消息时，可以为消息指定key，但是要求key要均匀，否则会出现kafka分区间数据不均衡。
​

### 解决方案

- 任务重新启动后直接消费最新的消息，对于"滞后"的历史数据采用离线程序进行"补漏"。
- 如下面图所示。创建新的topic并配置更多数量的分区，将积压消息的topic消费者逻辑改为直接把消息打入新的topic，将消费逻辑写在新的topic的消费者中。

​

![](https://gitee.com/littleeight/blog-images/raw/master/kafka%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/9.png)



- 对于重平衡导致的问题，不考虑人为的增加分区、消费者等情况，只考虑一些会让kafka误判的情况。
   - 设置**session.timout.ms大一些，保证zookeeper不容易判定消费者超时导致重平衡。**
   - **heartbeat.interval.ms，这个参数控制发送心跳的频率**，频率越高越不容易被误判，但也会消耗更多资源。
   - **max.poll.interval.ms**，我们都知道消费者poll数据后，需要一些处理，再进行拉取。如果两次拉取时间间隔超过这个参数设置的值，那么消费者就会被踢出消费者组。也就是说，拉取，然后处理，这个处理的时间不能超过max.poll.interval.ms这个参数的值。这个参数的默认值是5分钟，而如果消费者接收到数据后会执行耗时的操作，则应该将其设置得大一些。这里给出一个相对较为合理的配置，如下：
      - session.timout.ms：设置为6s
      - heartbeat.interval.ms：设置2s
      - max.poll.interval.ms：推荐为消费者处理消息最长耗时再加1分钟	
