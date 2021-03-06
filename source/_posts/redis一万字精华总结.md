---
title: 'redis一万字精华总结'
date: 2021-09-06 12:27:20
categories: 
  - 缓存
tags: 
  - little_eight
---
## redis的官网简介
&ensp;&ensp;&ensp;&ensp;Redis 是一个开放源码(BSD 许可)、内存中的数据结构存储，用作数据库、缓存和消息代理。Redis 提供数据结构，例如string字符串、hash散列、list列表、set集合、sorted set带有范围查询的排序集、bitmaps位图、hyperloglogs超级日志、geospatial indexes地理空间索引、 streams流。Redis 具有内置的复制、 Lua 脚本、 LRU 收回、事务和不同级别的磁盘持久性，并通过 Redis Sentinel 提供高可用性服务，并通过 Redis Cluster 提供自动分区。
&ensp;&ensp;&ensp;&ensp;您可以对这些类型运行原子操作，比如附加到字符串; 在散列中递增值; 将元素推入列表; 计算集合的交集、并集和差集; 或者获得排序集中排名最高的成员。
&ensp;&ensp;&ensp;&ensp;为了获得最佳性能，Redis 使用内存中的数据集。根据您的用例，您可以通过定期将数据集转储到磁盘或将每个命令附加到基于磁盘的日志中来持久化数据。如果您只需要一个功能丰富的网络化内存缓存，那么还可以禁用持久性。
&ensp;&ensp;&ensp;&ensp;Redis 还支持异步复制，具有非常快的非阻塞第一同步，自动重连接和部分重新同步的网络分割。
&ensp;&ensp;&ensp;&ensp;其他功能包括:
<!--more--> 
- [Transactions 交易](https://redis.io/topics/transactions)
- [Pub/Sub 发布/订阅](https://redis.io/topics/pubsub)
- [Lua scriptingLua 脚本](https://redis.io/commands/eval)
- [Keys with a limited time-to-live 有限时间的](https://redis.io/commands/expire)keys
- [LRU eviction of keysLRU 收回钥匙](https://redis.io/topics/lru-cache)
- [Automatic failover 自动故障转移](https://redis.io/topics/sentinel)

​

&ensp;&ensp;&ensp;&ensp;Redis 是用 ANSI c 编写的，在大多数 POSIX 系统中工作，比如 Linux、 * BSD 和 OS x，没有外部依赖性。Linux 和 OS x 是 Redis 开发和测试最多的两个操作系统，我们建议使用 Linux 进行部署。Redis 也许可以在诸如 SmartOS 这样的 solaris 衍生系统中工作，但是这种支持是最好的努力。没有对 Windows 构建的官方支持。
​

## redis的快
&ensp;&ensp;&ensp;&ensp;redis为什么能风靡全球，首要原因就是因为它快，在网络发展如此迅速的时代，时间就是金钱，一个良好的快速反馈时间是第一要素。那为什么redis能这么快？
### 纯内存存储
&ensp;&ensp;&ensp;&ensp;redis将所有数据放在内存中，非数据同步正常工作中，是不需要从磁盘读取数据的，0次IO。内存响应时间大约为100纳秒，所以理论上redis是可以达到100*1000qps的。
### 
### I/O多路复用
&ensp;&ensp;&ensp;&ensp;I/O 多路复用(select/poll/epoll)机制中多路是指多个连接，复用是指一个线程多次重复使用，也就是一个线程处理多个 IO 流，redis采用的是epoll，在 redis 只运行单线程的情况下，该机制允许内核中同时存在多个监听套接字和已连接套接字。内核会一直监听这些套接字上的连接请求或数据请求，一旦有请求到达就会交给 redis 线程处理，这样就实现了一个 redis 线程处理多个 IO 流的效果。看下图：
​

![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/1.png)

- 一个 socket 客户端与服务端连接时，会生成对应一个套接字描述符(套接字描述符是文件描述符的一种)，每一个 socket 网络连接其实都对应一个文件描述符。
- 多个客户端与服务端连接时，redis 使用 **「I/O 多路复用程序」** 将客户端 socket 对应的 FD 注册到监听列表(**一个队列**)中。当客服端执行 read、write 等操作命令时，I/O 多路复用程序会将命令封装成一个事件，并绑定到对应的 文件描述符（FD ） 上。
- **「文件事件处理器」**使用 I/O 多路复用模块同时监控多个 FD 的读写情况（如下图），当 accept、read、write等文件事件产生时，文件事件处理器就会回调 FD 绑定的事件处理器进行处理相关命令操作。
- 整个文件事件处理器是在单线程上运行的，但是通过 I/O 多路复用模块的引入，实现了同时对多个 FD 读写的监控，当其中一个 client 端达到写或读的状态，文件事件处理器就马上执行，从而就不会出现 I/O 堵塞的问题，提高了网络通信的性能。

![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/2.png)

### 单线程
&ensp;&ensp;&ensp;&ensp;单线程避免了线程上下文切换以及加锁释放锁带来的消耗，对于服务端开发来说，锁和线程切换通常是性能杀手。当然了，单线程也会有它的缺点，也是redis的噩梦：**阻塞**。如果执行一个命令过长，那么会造成其他命令的阻塞，对于Redis是十分致命的，所以Redis是面向快速执行场景的数据库。
&ensp;&ensp;&ensp;&ensp;这里需要提到的是，在redis4.0之后引入了多线程，像惰性删除，持久化、集群数据同步等操作，都是由额外的线程执行，而redis单线程是指**主线程专注于网络 IO 和键值对读写**。在redis6引入的多线程则是真正为了提高 I/O 的读写性能而引入的，它的主要实现思路是将主线程的 I/O 读写任务拆分给一组独立的子线程去执行，也就是说从 socket 中读数据和写数据不再由主线程负责，而是交给了多个子线程，这样就可以使多个 socket 的读写并行化了。这么做的原因就在于，虽然在 redis 中使用了 I/O 多路复用，但我们知道数据在内核态空间和用户态空间之间的拷贝是无法避免的，而数据的拷贝这一步是阻塞的，并且当数据量越大时拷贝所需要的时间就越多。所以多线程用于分摊同步读写 I/O 压力，从而提升 redis 的 qps。但是注意，redis 的命令本身依旧是由 redis 主线程串行执行的，只不过具体的读写操作交给独立的子线程去执行了，而这么做的好处就是不需要为 Lua 脚本、事务的原子性而额外开发多线程互斥机制，这样一来 redis 的线程模型实现起来就简单多了，因为和之前一样，所有的命令依旧是由主线程串行执行的，只不过具体的读写任务交给了子线程。下图是主线程跟io线程的交互情况：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/3.png)

&ensp;&ensp;&ensp;&ensp;redis6中，多线程机制默认是关闭的，如果想启动的话，需要修改 redis.conf 中的两个配置。第一个是设置io线程是否开启，第二个是设置其线程数。

      - io-thread-do-reads yes 
      - io-threads 3

&ensp;&ensp;&ensp;&ensp;除了redis之外，node.js、nginx也是单线程，他们都是服务器高性能的典范。
​

## redis的基础数据结构
### string
&ensp;&ensp;&ensp;&ensp;它是二进制安全的，可以存储图片或者序列化的对象，值最大存储为512M。底层用sds来使用，相对于c的原生字符串是char[]实现的好处之一是在获取长度时不需要遍历数据。
&ensp;&ensp;&ensp;&ensp;简单使用举例: set key value、get key等
&ensp;&ensp;&ensp;&ensp;应用场景：共享session、分布式锁，计数器、限流。
&ensp;&ensp;&ensp;&ensp;内部编码有3种，int（8字节长整型）/embstr（小于等于39字节字符串）/raw（大于39个字节字符串）
### hash
&ensp;&ensp;&ensp;&ensp;哈希类型是指v（值）本身又是一个键值对（k-v）结构
&ensp;&ensp;&ensp;&ensp;简单使用举例：hset key field value 、hget key field
&ensp;&ensp;&ensp;&ensp;内部编码：ziplist（压缩列表） 、hashtable（哈希表）
&ensp;&ensp;&ensp;&ensp;应用场景：缓存用户信息等。
**注意点**：如果开发使用hgetall，哈希元素比较多的话，可能导致redis阻塞，可以使用hscan。而如果只是获取部分field，建议使用hmget。
### 
list
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/4.png)

&ensp;&ensp;&ensp;&ensp;列表（list）类型是用来存储多个有序的字符串，一个列表最多可以存储2^32-1个元素。
&ensp;&ensp;&ensp;&ensp;简单实用举例： lpush key value [value ...] 、lrange key start end
&ensp;&ensp;&ensp;&ensp;内部编码：ziplist（压缩列表）、linkedlist（链表）
&ensp;&ensp;&ensp;&ensp;应用场景： 消息队列，文章列表。
​

### set
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/5.png)

&ensp;&ensp;&ensp;&ensp;集合（set）类型也是用来保存多个的字符串元素，但是不允许重复元素。
&ensp;&ensp;&ensp;&ensp;简单使用举例：sadd key element [element ...]、smembers key
&ensp;&ensp;&ensp;&ensp;内部编码：intset（整数集合）、hashtable（哈希表）
&ensp;&ensp;&ensp;&ensp;**注意点**：smembers和lrange、hgetall都属于比较重的命令，如果元素过多存在阻塞Redis的可能性，可以使用sscan来完成。
&ensp;&ensp;&ensp;&ensp;应用场景： 用户标签,生成随机数抽奖、社交需求。
​

### zset
&ensp;&ensp;&ensp;&ensp;已排序的字符串集合，同时元素不能重复。
&ensp;&ensp;&ensp;&ensp;简单格式举例：zadd key score member [score member ...]，zrank key member
&ensp;&ensp;&ensp;&ensp;底层内部编码：ziplist（压缩列表）、skiplist（跳跃表）
&ensp;&ensp;&ensp;&ensp;应用场景：排行榜，社交需求（如用户点赞）。
​

​

### bitmaps
&ensp;&ensp;&ensp;&ensp;用一个比特位来映射某个元素的状态，在redis中，它的底层是基于字符串类型实现的，可以把bitmaps成作一个以比特位为单位的数组。
### hyperloglogs
&ensp;&ensp;&ensp;&ensp;用来做基数统计算法的数据结构，如统计网站的浏览人数，帮程序自主去重。
### geospatial indexes
&ensp;&ensp;&ensp;&ensp;用来推算地理位置的信息，两地之间的距离，方圆几里的人等。
### streams
&ensp;&ensp;&ensp;&ensp;官方把它定义为：以更抽象的方式建模日志的数据结构。redis的streams主要是一个append only file的数据结构，至少在概念上它是一种在内存中表示的抽象数据类型，只不过它们实现了更强大的操作，以克服日志文件本身的限制。如果你了解MQ，那么可以把streams当做MQ。如果你还了解kafka，那么甚至可以把streams当做kafka。
&ensp;&ensp;&ensp;&ensp;另外，这个功能有点类似于redis以前的Pub/Sub，但是也有基本的不同：

- streams支持多个客户端（消费者）等待数据（Linux环境开多个窗口执行XREAD即可模拟），并且每个客户端得到的是完全相同的数据。
- Pub/Sub是发送忘记的方式，并且不存储任何数据；而streams模式下，所有消息被无限期追加在streams中，除非用于显示执行删除（XDEL）。
- streams的Consumer Groups也是Pub/Sub无法实现的控制方式。

## redis的底层数据结构
&ensp;&ensp;&ensp;&ensp;每次在Redis数据库中创建一个键值对时，至少会创建两个对象，一个是键对象，一个是值对象，而Redis中的每个对象都是由 redisObject 结构来表示：
```java
typedef struct redisObject{
     //类型
     unsigned type:4;
     //编码
     unsigned encoding:4;
     //指向底层数据结构的指针
     void *ptr;
     //引用计数
     int refcount;
     //记录最后一次被程序访问的时间
     unsigned lru:22;
}
```
&ensp;&ensp;&ensp;&ensp;其中type属性记录了对象的基础数据结构类型，也就是前面提到了五种：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add1.png)

&ensp;&ensp;&ensp;&ensp;对象的 prt 指针指向对象底层的数据结构，而数据结构由 encoding 属性来决定：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add2.png)

&ensp;&ensp;&ensp;&ensp;而每种类型的对象都至少使用了两种不同的编码：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add3.png)

### 字符串对象-string的构成
&ensp;&ensp;&ensp;&ensp;字符串是Redis最基本的数据类型，不仅所有key都是字符串类型，其它几种数据类型构成的元素也是字符串。注意字符串的长度不能超过512M。
&ensp;&ensp;&ensp;&ensp;字符串对象的编码可以是int、embstr、raw。
1、int 编码：保存的是可以用 long 类型表示的整数值。
2、embstr 编码：保存长度小于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）。
3、raw 编码：保存长度大于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）。

&ensp;&ensp;&ensp;&ensp;其中raw图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add4.png)


&ensp;&ensp;&ensp;&ensp;embstr图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add5.png)

&ensp;&ensp;&ensp;&ensp;embstr与raw都使用redisObject和sds保存数据，区别在于，embstr的使用只分配一次内存空间（因此redisObject和sds是连续的），而raw需要分配两次内存空间（分别为redisObject和sds分配空间）。因此与raw相比，embstr的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。而embstr的坏处也很明显，如果字符串的长度增加需要重新分配内存时，整个redisObject和sds都需要重新分配空间，因此redis中的embstr实现为只读。

&ensp;&ensp;&ensp;&ensp;当 int 编码保存的值不再是整数，或大小超过了long的范围时，自动转化为raw。
&ensp;&ensp;&ensp;&ensp;对于 embstr 编码，由于 Redis 没有对其编写任何的修改程序（embstr 是只读的），在对embstr对象进行修改时，都会先转化为raw再进行修改，因此，只要是修改embstr对象，修改后的对象一定是raw的，无论是否达到了44个字节。

### 列表对象-list的构成
&ensp;&ensp;&ensp;&ensp;list 列表，它是简单的字符串列表，按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边），它的底层实际上是个链表结构。
&ensp;&ensp;&ensp;&ensp;列表对象的编码可以是 ziplist(压缩列表) 和 linkedlist(双端链表)。 

&ensp;&ensp;&ensp;&ensp;其中ziplist图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add6.png)


&ensp;&ensp;&ensp;&ensp;linkedlist图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add7.png)

&ensp;&ensp;&ensp;&ensp;当同时满足下面两个条件时，使用ziplist（压缩列表）编码：
1、列表保存元素个数小于512个
2、每个元素长度小于64字节
&ensp;&ensp;&ensp;&ensp;不能满足这两个条件的时候使用 linkedlist 编码。
&ensp;&ensp;&ensp;&ensp;上面两个条件可以在redis.conf 配置文件中的 list-max-ziplist-value选项和 list-max-ziplist-entries 选项进行配置。

### 哈希对象-hash的构成
&ensp;&ensp;&ensp;&ensp;哈希对象的键是一个字符串类型，值是一个键值对集合。
&ensp;&ensp;&ensp;&ensp;哈希对象的编码可以是 ziplist 或者 hashtable。

&ensp;&ensp;&ensp;&ensp;其中ziplist图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add8.png)


&ensp;&ensp;&ensp;&ensp;hashtable图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add9.png)

&ensp;&ensp;&ensp;&ensp;hashtable 编码的哈希表对象底层使用字典数据结构，哈希对象中的每个键值对都使用一个字典键值对。
&ensp;&ensp;&ensp;&ensp;在前面介绍压缩列表时，我们介绍过压缩列表是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，相对于字典数据结构，压缩列表用于元素个数少、元素长度小的场景。其优势在于集中存储，节省空间。

&ensp;&ensp;&ensp;&ensp;和上面列表对象使用 ziplist 编码一样，当同时满足下面两个条件时，使用ziplist（压缩列表）编码：
1、列表保存元素个数小于512个
2、每个元素长度小于64字节
&ensp;&ensp;&ensp;&ensp;不能满足这两个条件的时候使用 hashtable 编码。第一个条件可以通过配置文件中的 set-max-intset-entries 进行修改。

### 集合对象-set的构成
&ensp;&ensp;&ensp;&ensp;集合对象 set 是 string 类型（整数也会转换成string类型进行存储）的无序集合。注意集合和列表的区别：集合中的元素是无序的，因此不能通过索引来操作元素；集合中的元素不能有重复。
&ensp;&ensp;&ensp;&ensp;集合对象的编码可以是 intset 或者 hashtable。

&ensp;&ensp;&ensp;&ensp;其中intset图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add10.png)


&ensp;&ensp;&ensp;&ensp;hashtable图示为：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add11.png)


&ensp;&ensp;&ensp;&ensp;当集合同时满足以下两个条件时，使用 intset 编码：
1、集合对象中所有元素都是整数
2、集合对象所有元素数量不超过512
&ensp;&ensp;&ensp;&ensp;不能满足这两个条件的就使用 hashtable 编码。第二个条件可以通过配置文件的 set-max-intset-entries 进行配置。

### 有序集合对象-zset的构成
&ensp;&ensp;&ensp;&ensp;和上面的集合对象相比，有序集合对象是有序的。与列表使用索引下标作为排序依据不同，有序集合为每个元素设置一个分数（score）作为排序依据。
&ensp;&ensp;&ensp;&ensp;有序集合的编码可以是 ziplist 或者 skiplist。

&ensp;&ensp;&ensp;&ensp;ziplist 编码的有序集合对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员，第二个节点保存元素的分值。并且压缩列表内的集合元素按分值从小到大的顺序进行排列，小的放置在靠近表头的位置，大的放置在靠近表尾的位置。
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add12.png)

&ensp;&ensp;&ensp;&ensp;skiplist 编码的有序集合对象使用 zet 结构作为底层实现，一个 zset 结构同时包含一个字典和一个跳跃表：
```java
typedef struct zset{
     //跳跃表
     zskiplist *zsl;
     //字典
     dict *dice;
} zset;
```
&ensp;&ensp;&ensp;&ensp;字典的键保存元素的值，字典的值则保存元素的分值；跳跃表节点的 object 属性保存元素的成员，跳跃表节点的 score 属性保存元素的分值。
&ensp;&ensp;&ensp;&ensp;这两种数据结构会通过指针来共享相同元素的成员和分值，所以不会产生重复成员和分值，造成内存的浪费。
&ensp;&ensp;&ensp;&ensp;说明：其实有序集合单独使用字典或跳跃表其中一种数据结构都可以实现，但是这里使用两种数据结构组合起来，原因是假如我们单独使用 字典，虽然能以 O(1) 的时间复杂度查找成员的分值，但是因为字典是以无序的方式来保存集合元素，所以每次进行范围操作的时候都要进行排序；假如我们单独使用跳跃表来实现，虽然能执行范围操作，但是查找操作有 O(1)的复杂度变为了O(logN)。因此Redis使用了两种数据结构来共同实现有序集合。

&ensp;&ensp;&ensp;&ensp;当有序集合对象同时满足以下两个条件时，对象使用 ziplist 编码：
1、保存的元素数量小于128；
2、保存的所有元素长度都小于64字节。
&ensp;&ensp;&ensp;&ensp;不能满足上面两个条件的使用 skiplist 编码。以上两个条件也可以通过Redis配置文件zset-max-ziplist-entries 选项和 zset-max-ziplist-value 进行修改。

## redis的过期策略
&ensp;&ensp;&ensp;&ensp;redis有三种过期策略。

- **定时过期**：每个设置过期时间的key都需要创建一个定时器，到过期时间就会立即对key进行清除。该策略可以立即清除过期的数据，对内存很友好；但是会占用大量的CPU资源去处理过期的数据，从而影响缓存的响应时间和吞吐量。
- **定期过期**：只有当访问一个key时，才会判断该key是否已过期，过期则清除。该策略可以最大化地节省CPU资源，却对内存非常不友好。极端情况可能出现大量的过期key没有再次被访问，从而不会被清除，占用大量内存。
- **惰性过期**：每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。该策略是前两者的一个折中方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得CPU和内存资源达到最优的平衡效果。expires字典会保存所有设置了过期时间的key的过期时间数据，其中，key是指向键空间中的某个键的指针，value是该键的毫秒精度的UNIX时间戳表示的过期时间。

​

&ensp;&ensp;&ensp;&ensp;redis同时使用的是定期过期跟惰性过期。


## redis的内存淘汰策略
&ensp;&ensp;&ensp;&ensp;内存并不是无限大的，当存储的容量达到一定限度时，redis就会采用内存淘汰策略来保护自己，主要有以下几种：

- **noeviction**：默认策略，当内存不足以容纳新写入数据时，新写入操作会报错。
- **volatile-lru**：当内存不足以容纳新写入数据时，从设置了过期时间的key中使用LRU（最近最少使用）算法进行淘汰。
- **allkeys-lru**：当内存不足以容纳新写入数据时，从所有key中使用LRU算法进行淘汰。
- **volatile-lfu**：4.0版本新增，当内存不足以容纳新写入数据时，在过期的key中，使用LFU算法进行删除key。 
- **allkeys-lfu**：4.0版本新增，当内存不足以容纳新写入数据时，从所有key中使用LFU算法进行淘汰。
- **volatile-random**：当内存不足以容纳新写入数据时，从设置了过期时间的key中，随机淘汰数据。
- **allkeys-random**：当内存不足以容纳新写入数据时，从所有key中随机淘汰数据。
- **volatile-ttl**：当内存不足以容纳新写入数据时，在设置了过期时间的key中，根据过期时间进行淘汰，越早过期的优先被淘汰。



## redis的持久化机制
&ensp;&ensp;&ensp;&ensp;redis是基于内存的，如果挂了数据便全部丢失，所以便需要做持久化，把数据存到磁盘中。redis有**rdb**跟**aof**两种机制。
### rdb
&ensp;&ensp;&ensp;&ensp;redis database，是把当前内存中的数据集快照写入磁盘，也就是 Snapshot 快照（数据库中所有键值对数据）。恢复时是将快照文件直接读到内存里。
&ensp;&ensp;&ensp;&ensp;rdb有2种触发机制，分别是自动触发和手动触发。

- **自动触发**，在 redis.conf 配置文件中的 SNAPSHOTTING 下，可配置相关策略。
   - **save：**这里是用来配置触发 Redis的 RDB 持久化条件，也就是什么时候将内存中的数据保存到硬盘。比如“save m n”。表示m秒内数据集存在n次修改时，自动触发bgsave（这个命令下面会介绍，手动触发RDB持久化的命令），当然如果你只是用Redis的缓存功能，不需要持久化，那么你可以注释掉所有的 save 行来停用保存功能。可以直接一个空字符串来实现停用：save ""。
   - **stop-writes-on-bgsave-error ：**默认值为yes。当启用了RDB且最后一次后台保存数据失败，Redis是否停止接收数据。这会让用户意识到数据没有正确持久化到磁盘上，否则没有人会注意到灾难（disaster）发生了。如果Redis重启了，那么又可以重新开始接收数据了。
   - **rdbcompression ；**默认值是yes。对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis会采用LZF算法进行压缩。如果你不想消耗CPU来进行压缩的话，可以设置为关闭此功能，但是存储在磁盘上的快照会比较大。
   - **rdbchecksum ：**默认值是yes。在存储快照后，我们还可以让redis使用CRC64算法来进行数据校验，但是这样做会增加大约10%的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能。
   - **dir：**设置快照文件的存放路径，这个配置项一定是个目录，而不能是文件名。默认是和当前配置文件保存在同一目录。

&ensp;&ensp;&ensp;&ensp;也就是说通过在配置文件中配置的 save 方式，当实际操作满足该配置形式时就会进行 RDB 持久化，将当前的内存快照保存在 dir 配置的目录中，文件名由配置的 dbfilename 决定。
​


- **手动触发**
   - **save **该命令会阻塞当前Redis服务器，执行save命令期间，Redis不能处理其他命令，直到RDB过程完成为止。
   - **bgsave **执行该命令时，Redis会在后台异步进行快照操作，快照同时还可以响应客户端请求。具体操作是Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短。

&ensp;&ensp;&ensp;&ensp;基本上 Redis 内部所有的RDB操作都是采用 bgsave 命令。
​

#### rdb的优缺点
1、rdb是一个非常紧凑的文件，保存了redis在某个时间点的数据集，非常适合备份，体积比aof小，因为是数据的快照，基本上就是数据的复制，不用重新读取再写入内存。
2、rdb的工作原理是父进程在保存文件就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作,这样可以最大化提升redis的性能，**但是当数据比较大时这个fork进程可能会非常耗时，造成redis阻塞。**
3、rdb在恢复大数据集时比aof慢。
4、因为是根据时间来备份的，所以会有丢失数据的风险。
​

### aof
&ensp;&ensp;&ensp;&ensp;append only file，将写操作追加到文件中，AOF 日志是写后日志，“写后”的意思是 redis 是先执行命令，把数据写入内存后，然后才记日志；里面记录的是指令执行的步骤，非常详细，描绘出了数据的变化过程。
&ensp;&ensp;&ensp;&ensp;在 redis.conf 配置文件中的 SNAPSHOTTING 下，可配置相关策略。

- **appendonly**  no 是否开启AOF机制，yes 代表开启
- **appendfilename**  "appendonly.aof"   aof文件名
- **appendfsync** xxx   aof持久化策略的配置， xxx为alaways表示表示不执行同步，由操作系统自己选择时间保证数据同步到磁盘，速度最快；xxx为everysec表示每一秒执行一次同步，可能会导致丢失这1s数据；no表示每次写入内存后都执行同步，以保证数据同步到磁盘。
- **no-appendfsync-on-rewrite**  no  是否开启重写(当aof文件的大小超过所设定的阈值时，redis就会对aof文件的内容压缩。)
- **auto-aof-rewrite-percentage ** 100   当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写
- **auto-aof-rewrite-min-size**  64mb    设置允许重写的最小aof文件大小
- **aof-use-rdb-preamble**  no      混合使用 aof和rdb的开关
### 
#### aof的优缺点
1、可最大限度地保证数据的完整性。
2、重写机制让日志文件更小。
3、因为记录的是执行过程，所以文件会比rdb大许多。
​

## redis的高可用
&ensp;&ensp;&ensp;&ensp;高可用的方案一般都是集群，redis有三种集群方式：**主从模式，哨兵模式，集群模式**。
​

### 主从模式 master-slave
&ensp;&ensp;&ensp;&ensp;主从模式中，Redis部署了多台机器，有主节点，负责读写操作，有从节点，只负责读操作。从节点的数据来自主节点，实现原理就是**主从复制机制，**基本的步骤如下（蓝色代表slave发给master的命令，红色相反）**：**


- slave发送psync2命令到master。（命令：psync2 <runid> <offset>）
- master接收到SYNC命令后，执行bgsave命令，生成RDB全量文件,并生成缓冲区（缓冲复制先进先出队列，默认1M）记录从现在开始执行的所有写命令。（命令： fullresync <runid> <offset>）
- master执行完bgsave后，向所有slave发送RDB快照文件。
- slave收到RDB快照文件后，清空自己的数据，然后载入、解析收到的快照。
- master快照发送完毕后，也就是**全量复制**结束后，会开始**增量复制**，向slave发送缓冲区中的写命令。
- slave接受命令请求，并执行来自master缓冲区的写命令。
- 后续slave只需要携带id跟offset(上次复制的偏移量)发给master，便可进行**增量复制**。如果存在就会发送continue给slave，如果不存在就会执行全量复制。(命令 continue <runid> <offset>)



​

### 哨兵模式 sentinel
&ensp;&ensp;&ensp;&ensp;主从模式中，一旦主节点由于故障不能提供服务，需要人工将从节点晋升为主节点，同时还要通知应用方更新主节点地址。显然，多数业务场景都不能接受这种故障处理方式。redis从2.8开始正式提供了Redis Sentinel（哨兵）架构来解决这个问题。
**哨兵模式**，由一个或多个Sentinel实例组成的Sentinel系统，它可以监视所有的redis主节点和从节点，并在被监视的主节点进入下线状态时，**自动将下线主服务器属下的某个从节点升级为新的主节点**。但是呢，一个哨兵进程对redis节点进行监控，就可能会出现问题（**单点问题**），因此，可以使用多个哨兵来进行监控redis节点，并且各个哨兵之间还会进行监控。下图是整体架构：
![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/6.png)


&ensp;&ensp;&ensp;&ensp;简单来说，哨兵模式就三个作用：

- 发送命令，等待Redis服务器（包括主服务器和从服务器）返回监控其运行状态；
- 哨兵监测到主节点宕机，会自动将从节点切换成主节点，然后通过发布订阅模式通知其他的从节点，修改配置文件，让它们切换主机；
- 哨兵之间还会相互监控，从而达到高可用。



&ensp;&ensp;&ensp;&ensp;哨兵的工作模式如下：

1. 每个哨兵以每秒钟一次的频率向它所知的master，slave以及其他哨兵实例发送一个 PING命令。
1. 如果一个实例（instance）距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被哨兵标记为主观下线。
1. 如果一个master被标记为主观下线，则正在监视这个master的所有哨兵要以每秒一次的频率确认master的确进入了主观下线状态。
1. 当有足够数量的哨兵（大于等于配置文件指定的值）在指定的时间范围内确认master的确进入了主观下线状态， 则master会被标记为客观下线。
1. 在一般情况下， 每个哨兵会以每10秒一次的频率向它已知的所有master，slave发送 INFO 命令,当master被哨兵标记为客观下线时，哨兵向下线的 master的所有 slave发送 INFO 命令的频率会从 10 秒一次改为每秒一次
1. 若没有足够数量的哨兵同意master已经下线， master的客观下线状态就会被移除；若master重新向哨兵的 PING 命令返回有效回复， master的主观下线状态就会被移除。



&ensp;&ensp;&ensp;&ensp;当master被判断客观下线以后，各个哨兵节点会进行协商，选举出一个领导者哨兵节点，并由该领导者节点对其进行故障转移操作。监视该主节点的所有哨兵都有可能被选为领导者，选举使用的算法是 Raft 算法，Raft 算法的基本思路是先到先得：即在一轮选举中，哨兵 A 向 B 发送成为领导者的申请，如果 B 没有同意过其他哨兵，则会同意 A 成为领导者。
选&ensp;&ensp;&ensp;&ensp;举出的领导者哨兵，开始进行**故障转移操作**，该操作大体可以分为 3 个步骤：

   1. 在从节点中选择新的主节点：选择的原则是，首先过滤掉不健康的从节点，然后选择优先级最高的从节点(由 slave-priority 指定)。 如果优先级无法区分，则选择复制偏移量最大的从节点；如果仍无法区分，则选择 runid 最小的从节点。
   1. 更新主从状态：通过 slaveof no one 命令，让选出来的从节点成为主节点；并通过 slaveof 命令让其他节点成为其从节点。
   1. 将已经下线的主节点设置为新的主节点的从节点，当它重新上线后，它会成为新的主节点的从节点。


### 集群模式 cluster
&ensp;&ensp;&ensp;&ensp;哨兵模式基于主从模式，实现读写分离，它还可以自动切换，系统可用性更高。但是它每个节点存储的数据是一样的，浪费内存，并且不好在线扩容。 因此，cluster集群应运而生，它在redis3加入的，实现了redis的**分布式存储**。对数据进行分片，也就是说**每台redis节点上存储不同的内容**，来解决在线扩容的问题。并且它也提供**复制**和**故障转移**的功能。
&ensp;&ensp;&ensp;&ensp;cluster集群通过Gossip协议进行通信，节点之前不断交换信息，交换的信息内容包括节点出现故障、新节点加入、主从节点变更信息、slot信息等等。常用的Gossip消息分为4种，分别是：ping、pong、meet、fail。

- **meet消息**：通知新节点加入。消息发送者通知接收者加入到当前集群，meet消息通信正常完成后，接收节点会加入到集群中并进行周期性的ping、pong消息交换。
-  **ping消息**：集群内交换最频繁的消息，集群内每个节点每秒向多个其他节点发送ping消息，用于检测节点是否在线和交换彼此状态信息。
-  **pong消息**：当接收到ping、meet消息时，作为响应消息回复给发送方确认消息正常通信。pong消息内部封装了自身状态数据。节点也可以向集群内广播自身的pong消息来通知整个集群对自身状态进行更新。
-  **fail消息**：当节点判定集群内另一个节点下线时，会向集群内广播一个fail消息，其他节点接收到fail消息之后把对应节点更新为下线状态。



#### 数据存储
&ensp;&ensp;&ensp;&ensp;cluster如何做到每个节点存储不同数据的呢，它采用的是**hash slot插槽算法，插槽算法**把整个数据库被分为16384个slot（槽），每个进入redis的键值对，根据key进行散列，分配到这16384插槽中的一个。使用的哈希映射也比较简单，用CRC16算法计算出一个16 位的值，再对16384取模。数据库中的每个键都属于这16384个槽的其中一个，集群中的每个节点都可以处理这16384个槽。
&ensp;&ensp;&ensp;&ensp;集群中的每个节点负责一部分的hash槽，比如当前集群有A、B、C个节点，每个节点上的哈希槽数 =16384/3，那么就有：

- 节点A负责0~5460号哈希槽
- 节点B负责5461~10922号哈希槽
- 节点C负责10923~16383号哈希槽

&ensp;&ensp;&ensp;&ensp;如果新增节点，那就会把其他节点的头部部分哈希槽一起平分给新节点，如果删除节点的话就平分到其他节点。
	设计成16384个槽点是因为考虑到节点数不太可能超过1000，并且槽点越小，其压缩率就越高。
#### 复制
&ensp;&ensp;&ensp;&ensp;cluster集群引入了主从复制，一个主节点对应一个或者多个从节点。当其它主节点 ping 一个主节点 A 时，如果半数以上的主节点与 A 通信超时，那么认为主节点 A 宕机了。如果主节点宕机时，就会启用从节点。
&ensp;&ensp;&ensp;&ensp;当集群内节点出现故障时，通过**故障转移**，以保证集群正常对外提供服务。
&ensp;&ensp;&ensp;&ensp;redis集群通过ping/pong消息，实现故障发现。这个环境包括**主观下线和客观下线**。

- **主观下线：** 某个节点认为另一个节点不可用，即下线状态，这个状态并不是最终的故障判定，只能代表一个节点的意见，可能存在误判情况。

![](https://gitee.com/littleeight/blog-images/raw/master/redis%E7%9A%84%E4%B8%80%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/7.png)


- **客观下线：** 指标记一个节点真正的下线，集群内多个节点都认为该节点不可用，从而达成共识的结果。如果是持有槽的主节点故障，需要为该节点进行故障转移。
- 假如节点A标记节点B为主观下线，一段时间后，节点A通过消息把节点B的状态发到其它节点，当节点C接受到消息并解析出消息体时，如果发现节点B的pfail状态时，会触发客观下线流程；
- 当下线为主节点时，此时Redis Cluster集群为统计持有槽的主节点投票，看投票数是否达到一半，当下线报告统计数大于一半时，被标记为**客观下线**状态。
- ​**故障恢复**：故障发现后，如果下线节点的是主节点，则需要在它的从节点中选一个替换它，以保证集群的高可用。流程如下：
   - 资格检查：检查从节点是否具备替换故障主节点的条件。
   - 准备选举时间：资格检查通过后，更新触发故障选举时间。
   - 发起选举：到了故障选举时间，进行选举。
   - 选举投票：只有持有槽的**主节点**才有票，从节点收集到足够的选票（大于一半），触发**替换主节点操作**
   - **​**

## mysql与redis保证双写一致性

### 延时双删
1. 先删除缓存
1. 再更新数据库
1. 休眠一会（比如1秒），再次删除缓存。

​

&ensp;&ensp;&ensp;&ensp;这种方案还算可以，只有休眠那一会（比如就那1秒），可能有脏数据，一般业务也会接受的。但是如果**第二次删除缓存失败**呢？缓存和数据库的数据还是可能不一致。
​

### 删除缓存重试机制

1. 写请求更新数据库
1. 缓存因为某些原因，删除失败
1. 把删除失败的key放到消息队列
1. 消费消息队列的消息，获取要删除的key
1. 重试删除缓存操作

​

### 读取binlog异步删除缓存

- 可以使用阿里的canal将binlog日志采集发送到MQ队列里面
- 然后通过ACK机制确认处理这条更新消息，删除缓存，保证数据缓存一致性

​

## redis的分布式锁
&ensp;&ensp;&ensp;&ensp;这个可用redisson实现，具体可查看[redisson分布式锁详解](https://www.yuque.com/xiaoba-t6gyt/mtab2v/hv2phx)。


## redis对于缓存的三大问题的处理
### 缓存雪崩
&ensp;&ensp;&ensp;&ensp;缓存雪崩是指在某一个时间段，缓存集中过期失效。此刻无数的请求直接绕开缓存，直接请求数据库。
&ensp;&ensp;&ensp;&ensp;造成缓存雪崩的原因，有以下两种：**redis宕机**、**大部分数据失效**。
&ensp;&ensp;&ensp;&ensp;对于缓存雪崩的解决方案有以下两种：
- 搭建高可用的集群，防止单机的redis宕机。
- 设置不同的过期时间，防止同一时间内大量的key失效。

​

### 缓存穿透
&ensp;&ensp;&ensp;&ensp;缓存穿透是指查询一条数据库和缓存都没有的一条数据，就会一直查询数据库，对数据库的访问压力就会增大；
&ensp;&ensp;&ensp;&ensp;解决方案：
- 缓存空对象
- 布隆过滤器

​

### 缓存击穿
&ensp;&ensp;&ensp;&ensp;是指一个key非常热点，在不停的扛着大并发，**大并发**集中对这一个点进行访问，当这个key在失效的瞬间，持续的**大并发**就穿破缓存，直接请求数据库，瞬间对数据库的访问压力增大。
解决方案：
- 将数据热点数据设置成永久的，不设置失效时间。
- 集群扩容，增加分片副本，均衡读流量。
- 使用分布式锁，a、当缓存不命中时，在查询数据库前使用redis分布式锁，使用查询的key值作为锁条件；b、获取锁的线程在查询数据库前，再查询一次缓存。这样做是因为高并发请求获取锁的时候造成排队，但第一次进来的线程在查询完数据库后会写入缓存，之后再获得锁的线程直接查询缓存就可以获得数据；c、读取完数据后释放分布式锁。
