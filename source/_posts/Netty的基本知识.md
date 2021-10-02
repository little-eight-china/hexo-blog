---
title: 'Netty的基本知识'
date: 2021-07-02 17:22:00
categories: 
  - java第三方框架
tags: 
  - little_eight
---

## 什么是netty
&ensp;&ensp;&ensp;&ensp;netty是一个 nio 客户机服务器框架，它支持快速简单地开发网络应用程序，如协议服务器和客户机。它极大地简化了网络编程，如 TCP 和 UDP 套接字服务器。经过多年的发展它已成为构建一个java网络生态的首选架构，在一些开源的项目中可见。
&ensp;&ensp;&ensp;&ensp;netty是一个异步、事件驱动的网络框架，整个netty的api都是异步设计，对网络应用来讲，io一般是性能的瓶颈，使用异步io可以较大程度上提高程序性能。
&ensp;&ensp;&ensp;&ensp;下面是官网的netty框架的图：
![](https://gitee.com/littleeight/blog-images/raw/master/netty%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%9F%A5%E8%AF%86/1.png)
​<!--more-->
&ensp;&ensp;&ensp;&ensp;有个有趣的一点是，由于netty5针对旧版本，复杂度增加了但是性能层面上并没有明显的性能优势，于是作者废弃了netty5。


## 为什么不使用java的nio
&ensp;&ensp;&ensp;&ensp;netty的底层就是基于jdk的nio来实现的，那我们干嘛不直接使用jdk的nio呢？**因为作者吃饱了撑着没事干。**
&ensp;&ensp;&ensp;&ensp;当然以上的原因是开玩笑的，使用netty的原因可以包括为2个大类，做的更多与做的更好。
​

### 规避java的nio的epoll bug
&ensp;&ensp;&ensp;&ensp;jdk的nio的**异常唤醒空转导致cpu100%**，官方在6670302-BUG页面上好像并不认为是jdk的bug，也没给出具体原因，而把原因归结为Linux Kernel 2.4版本的bug(JDK-6481709)。​
&ensp;&ensp;&ensp;&ensp;官方把问题归结于于linux的epoll(显然是被甩锅了)。如果一个socket文件描述符，注册的事件集合码为0，然后连接突然被对端中断，那么epoll会被POLLHUP或者有可能是POLLERR事件给唤醒，并返回到事件集中去。这意味着，Selector会被唤醒，即使对应的channel兴趣事件集是0，并且返回的events事件集合也是0。
&ensp;&ensp;&ensp;&ensp;简而言之就是，jdk认为linux的epoll告诉我事件来了，但是jdk没有拿到任何事件(READ、WRITE、CONNECT、ACCPET)。但此时select()方法不再选择阻塞了，而是选择返回了0。
&ensp;&ensp;&ensp;&ensp;后期版本对于该问题也只是减少发生的频率并没有从根本上解决。
&ensp;&ensp;&ensp;&ensp;netty就很积极地面对了这个问题。在NioEventLoop里设定了当阻塞时间小于 timeoutMillis，且select 执行次数 > _SELECTOR_AUTO_REBUILD_THRESHOLD _阈值（默认 512），认为发生了 epoll 空轮询。因为阻塞时间无法做到很精准，所以若某次阻塞时间大于等于 timeoutMillis 立刻重置 selectCnt 为 1，即需要 连续 512 次 selector.select(timeoutMillis) 阻塞时间都小于 timeoutMillis 才认为发生了 epoll 空轮询。然后netty通过重建 Selector 解决 epoll bug，新建selector，将旧的selector上的channel全部注册到新的selector上，然后关闭旧的。下面是2段关键代码的判断：


``` JAVA
private int select(long deadlineNanos) throws IOException {
    if (deadlineNanos == _NONE_) {
        return selector.select();
    }
    _// Timeout will only be 0 if deadline is within 5 microsecs
    _long timeoutMillis = _deadlineToDelayNanos_(deadlineNanos + 995000L) / 1000000L;
    return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);
}
```
​

``` JAVA
// _SELECTOR_AUTO_REBUILD_THRESHOLD 默认512_ `
if (_SELECTOR_AUTO_REBUILD_THRESHOLD _> 0 &&
        selectCnt >= _SELECTOR_AUTO_REBUILD_THRESHOLD_) {
    _// The selector returned prematurely many times in a row.
    // Rebuild the selector to work around the problem.
    logger_.warn("Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
            selectCnt, selector);`
// 重建selector
    rebuildSelector();
    return true;
}
```
​

### 规避tcp的IP_TOS参数的使用时抛出异常
&ensp;&ensp;&ensp;&ensp;netty会判断当option里有配置IP_TOS时，会选择返回false，也就是不支持该功能来规模异常。
​

### 更多有用的配置
&ensp;&ensp;&ensp;&ensp;比如经典的tcp的keepalive，netty提供了可开启keepalive的参数，并且增加了idle监测配合keepalive的使用。
&ensp;&ensp;&ensp;&ensp;当启用( 默认关闭) keepalive 时，tpc在连接没有数据通过的7200秒后发送keepalive消息，当探测没有确认按75秒的重试频率重发，一直发 9个探测包都没有确认，连接失效。所以总耗时一般为: 2小时11分钟(7200秒+xxs)。
&ensp;&ensp;&ensp;&ensp;而idle监测是一种诊断逻辑，根据诊断结果来做出不同的行为，比如进行keepalive操作，或者是直接关闭该连接。
## netty的快
### reactor模式
&ensp;&ensp;&ensp;&ensp;netty采用了nio的reactor模式(对于oio也是支持的，只不过标识了@Deprecated给相应的类，说明未来趋势是nio), 最大程度**减免了各个客户端连接与读写之间切换所花费的时间**。
&ensp;&ensp;&ensp;&ensp;reactor模式中定义了三种角色：

- reactor：负责监听跟分配事件，将i/o事件分派给对应的handle，新的事件包括连接建立就绪、读就绪、写就绪等。
- acceptor：多路复用器，处理客户端新连接，并分派请求到处理器链中。
- handle：事件处理器，将自身与事件绑定，执行异步读写任务。



&ensp;&ensp;&ensp;&ensp;常见的reactor模式有三种实现方式：
#### reactor单线程模式
&ensp;&ensp;&ensp;&ensp;reactor线程是个多面手，负责多路分离套接字，acceptor新连接，并分派请求到处理器链中。该模型适用于处理器链中业务处理组件能快速完成的场景。不过这种单线程模型不能充分利用多核资源，所以实际使用的不多。
&ensp;&ensp;&ensp;&ensp;netty中怎么实现：当NioEventLoopGroup只有bossGroup，而bossGroup只有一个线程，并且ServerBootstrap只包含bossGroup时，就是这个模式了。
![](https://gitee.com/littleeight/blog-images/raw/master/netty%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%9F%A5%E8%AF%86/2.png)


#### 非主从reactor多线程模式
&ensp;&ensp;&ensp;&ensp;该模型在handle采用了线程池，利用多个线程来执行读写任务。
&ensp;&ensp;&ensp;&ensp;netty中怎么实现：当NioEventLoopGroup只有bossGroup，而bossGroup有多个线程，并且ServerBootstrap只包含一个bossGroup时，就是这个模式了。
![](https://gitee.com/littleeight/blog-images/raw/master/netty%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%9F%A5%E8%AF%86/3.png)


#### 主从reactor多线程模式
&ensp;&ensp;&ensp;&ensp;比起非主从reactor多线程模式，它是将reactor分成两部分，mainReactor负责监听并accept新连接，然后将建立的socket通过acceptor分派给subReactor。subReactor负责多路分离已连接的socket，读写网络数据；业务处理功能，其交给worker线程池完成。通常，subReactor个数上可与CPU个数等同。
&ensp;&ensp;&ensp;&ensp;netty中怎么实现：当NioEventLoopGroup有bossGroup跟workerGroup，并且ServerBootstrap包含这2个group时，就是这个模式了。
![](https://gitee.com/littleeight/blog-images/raw/master/netty%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%9F%A5%E8%AF%86/4.png)


### 尽可能少的内存移动
&ensp;&ensp;&ensp;&ensp;众所周知，一个人搬东西从一个地方到另一个地方，东西的数量与行走的路线是决定完成搬迁花费的时间的重要因素。网络传输一样的道理，netty在一些地方尽量做到了用最少的内存，走最短的路径干最多的事情。下面举例比较典型的一些例子：
#### ByteToMessageDecoder
&ensp;&ensp;&ensp;&ensp;在传输过程中，我们需要解码器帮我们得到一个完整的数据包，这样的话就可能会有一个处理数据叠加的过程。
&ensp;&ensp;&ensp;&ensp;而这个类提供了两种实现方式**MERGE_CUMULATOR**跟**COMPOSITE_CUMULATOR**，前者使用内存复制，后者使用 CompositeByteBuf ，通过组合新输入的 ByteBuf 对象来实现数据叠加，从而避免内存拷贝。
#### DefaultFileRegion
&ensp;&ensp;&ensp;&ensp;netty传输文件的时候没有使用 ByteBuf 进行向 channel 中写入数据，而使用的 FileRegion。其中默认的实现DefaultFileRegion里的transferTo方法是通过使用java的nio的FileChannel的transferTo实现关键逻辑的，而这里实现了零拷贝复制，减少了io复制的次数来加快传输速度。

### 控制锁的范围
&ensp;&ensp;&ensp;&ensp;netty尽量减少了并发时对锁的范围的大小，以此来加快速度。举一个例子：
#### ServerBootstrap
&ensp;&ensp;&ensp;&ensp;在该类的childOption中，并不是对整个方法进行加锁，而是对关键的操作对象childOptions进行加锁，这样缩小了锁的范围可优化并发时线程等待时间。
``` JAVA
public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value) {
    ObjectUtil._checkNotNull_(childOption, "childOption");
    synchronized (childOptions) {
        if (value == null) {
            childOptions.remove(childOption);
        } else {
            childOptions.put(childOption, value);
        }
    }
    return this;
}
```


## tcp的粘包半包
&ensp;&ensp;&ensp;&ensp;由于tcp是一个流式协议，消息无边界，粘包半包是不可避免的问题。
&ensp;&ensp;&ensp;&ensp;粘包就是接收端读取时，多个发送过来的数据包粘到了一起，半包就是一个完整的数据包被分成了2部分2次发送。例如：
1、服务端一次接收到了两个数据包，D1和D2粘合在一起，被称为TCP粘包；
2、服务端分两次读取到了两个数据包，第一次读取到了完整的D1包和D2包的部分内容，
3、第二次读取到了D2包的剩余内容，这被称为TCP拆包
4、服务端分两次读取到了两个数据包，第一次读取到了D1包的部分内容D1_1，第二次读取到了D1包的剩余内容D1_2和D2包的整包。
​

### 解决思路
1、tcp连接改成短连接，一个请求一个短连接，建立连接到释放连接之间的信息即为传输信息。
2、封装成帧。基本思路是，在接收端，需要根据自定义协议来，来读取底层的数据包，重新组装我们应用层的数据包，这个过程通常在接收端称为**拆包**。

   a. 接收端应用层不断从底层的TCP 缓冲区中读取数据。
   b. 每次读取完，判断一下是否为一个完整的应用层数据包。如果是，上层应用层数据包读取完成。
   c. 如果不是，那就保留该数据在应用层缓冲区，然后继续从 TCP 缓冲区中读取，直到得到一个完整的应用层数据包为止。
   d. 至此，半包问题得以解决。
   e. 如果从TCP底层读到了多个应用层数据包，则将整个应用层缓冲区，拆成一个一个的独立的应用层数据包，返回给调用程序。
   f. 至此，粘包问题得以解决。

​

### 解决方案
&ensp;&ensp;&ensp;&ensp;netty提供了4种解码器（基类都是ByteToMessageDecoder）来解决，分别如下：
1、固定长度的拆包器 FixedLengthFrameDecoder，每个应用层数据包的都拆分成都是固定长度的大小，通过在包头增加消息体长度的解码器，解析数据时首先获取首部长度，然后定长读取socket中的数据。这种方式会导致空间浪费，不建议。
2、行拆包器 LineBasedFrameDecoder，每个应用层数据包，都以换行符作为分隔符，进行分割拆分。换行符解码器，报文尾部增加固定换行符rn，解析数据时以换行符作为报文结尾。
3、分隔符拆包器 DelimiterBasedFrameDecoder，每个应用层数据包，都通过自定义的分隔符，进行分割拆分。分隔符解码器，使用特定分隔符作为报文的结尾，解析数据时以定义的分隔符作为报文结尾
4、基于数据包长度的拆包器 LengthFieldBasedFrameDecoder，将应用层数据包的长度，作为接收端应用层数据包的拆分依据。按照应用层数据包的大小，拆包。这个拆包器，有一个要求，就是应用层协议中包含数据包的长度。定长解码器，这个最简单，消息体固定长度，解析数据时按长度读取即可。


## 核心类
​

&ensp;&ensp;&ensp;&ensp;一个netty程序开始于Bootstrap类，Bootstrap类是netty提供的一个可以通过简单配置来设置或"引导"程序的一个很重要的类。netty中设计了Handlers来处理特定的"event"和设置netty中的事件，从而来处理多个协议和数据。事件可以描述成一个非常通用的方法，因为你可以自定义一个handler,用来将Object转成byte[]或将byte[]转成Object；也可以定义个handler处理抛出的异常。 
&ensp;&ensp;&ensp;&ensp;你会经常编写一个实现ChannelInboundHandler的类，ChannelInboundHandler是用来接收消息，当有消息过来时，你可以决定如何处理。当程序需要返回消息时可以在ChannelInboundHandler里write/flush数据。可以认为应用程序的业务逻辑都是在ChannelInboundHandler中来处理的，业务罗的生命周期在ChannelInboundHandler中。 
&ensp;&ensp;&ensp;&ensp;netty连接客户端端或绑定服务器需要知道如何发送或接收消息，这是通过不同类型的handlers来做的，多个Handlers是怎么配置的？netty提供了ChannelInitializer类用来配置Handlers。ChannelInitializer是通过ChannelPipeline来添加ChannelHandler的，如发送和接收消息，这些Handlers将确定发的是什么消息。ChannelInitializer自身也是一个ChannelHandler，在添加完其他的handlers之后会自动从ChannelPipeline中删除自己。 
&ensp;&ensp;&ensp;&ensp;所有的netty程序都是基于ChannelPipeline。ChannelPipeline和EventLoop和EventLoopGroup密切相关，因为它们三个都和事件处理相关，所以这就是为什么它们处理IO的工作由EventLoop管理的原因。 
&ensp;&ensp;&ensp;&ensp;Netty中所有的IO操作都是异步执行的，例如你连接一个主机默认是异步完成的；写入/发送消息也是同样是异步。也就是说操作不会直接执行，而是会等一会执行，因为你不知道返回的操作结果是成功还是失败，但是需要有检查是否成功的方法或者是注册监听来通知；Netty使用Futures和ChannelFutures来达到这种目的。Future注册一个监听，当操作成功或失败时会通知。ChannelFuture封装的是一个操作的相关信息，操作被执行时会立刻返回ChannelFuture。


### Bootstrap、ServerBootstrap
      相当于客户端与服务端。
&ensp;&ensp;&ensp;&ensp;Bootstrap和ServerBootstrap之间的差异： 
&ensp;&ensp;&ensp;&ensp;Bootstrap用来连接远程主机，有1个EventLoopGroup,专门处理连接。
&ensp;&ensp;&ensp;&ensp;ServerBootstrap用来绑定本地端口，有2个EventLoopGroup，一个处理客户端的新连接，一个处理与客户端的交互。前者会轮询将连接交给后者的一个NioEventLoop处理。
​

&ensp;&ensp;&ensp;&ensp;netty的启动流程中，涉及到多个操作，比如register、bind、注册对应事件等，为了不影响main线程执行，这些工作以task的形式提交给NioEventLoop，由NioEventLoop来执行这些task，也就是register、bind、注册事件等操作。


​

### EventLoop、EventLoopGroup 
​

&ensp;&ensp;&ensp;&ensp;EventLoopGroup中可能包含了多个EventLoop，EventLoop是一个Reactor模型的事件处理器，一个EventLoop对应一个线程，其内部会维护一个selector和taskQueue（fifo的队列），负责处理客户端请求和内部任务，内部任务如ServerSocketChannel注册和ServerSocket绑定操作等。
&ensp;&ensp;&ensp;&ensp;IO事件和内部任务执行时间百分比通过ioRatio来调节，ioRatio表示执行IO时间所占百分比。任务包括普通任务和已经到时的延迟任务，延迟任务存放到一个优先级队列PriorityQueue中，执行任务前从PriorityQueue读取所有到时的task，然后添加到taskQueue中，最后统一执行task。
​

### Channel
&ensp;&ensp;&ensp;&ensp;在nio网络编程模型中, 服务端和客户端进行IO数据交互(得到彼此推送的信息)的媒介就是Channel。netty的channel包含了以下信息：

- id
- 可能存在的parent Channel
- 管道 pepiline
- 用于数据读写的unsafe内部类
- 关联上相伴终生的NioEventLoop

​

&ensp;&ensp;&ensp;&ensp;Channel提供了很多方法，如下列表： 

- eventLoop()，返回分配给Channel的EventLoop 
- pipeline()，返回分配给Channel的ChannelPipeline 
- isActive()，返回Channel是否激活，已激活说明与远程连接对等 
- localAddress()，返回已绑定的本地SocketAddress 
- remoteAddress()，返回已绑定的远程SocketAddress 
- write()，写数据到远程客户端，数据通过ChannelPipeline传输过去



### ChannelPipeline 
​

&ensp;&ensp;&ensp;&ensp;每个channel内部都会持有一个ChannelPipeline对象pipeline，ChannelPipeline 提供了一个容器给 ChannelHandler 链并提供了一个API 用于管理沿着链入站和出站事件的流动。pipeline默认实现DefaultChannelPipeline内部维护了一个DefaultChannelHandlerContext链表。
&ensp;&ensp;&ensp;&ensp;channel的读写操作都会走到DefaultChannelPipeline中，当channel完成register、active、read、readComplete等操作时，会触发pipeline的相应方法。
&ensp;&ensp;&ensp;&ensp;当channel注册到selector后，触发pipeline的fireChannelRegistered方法；
&ensp;&ensp;&ensp;&ensp;当channel是可用时，触发pipeline的fireChannelActive方法。（fireChannelActive触发一般是在fireChannelRegistered之后触发的）；
&ensp;&ensp;&ensp;&ensp;当客户端发送数据时，触发pipeline的fireChannelRead方法，触发pipeline的fireChannelRead方法之后会触发pipeline的fireChannelReadComplete方法。
​

### Future、Promise
&ensp;&ensp;&ensp;&ensp;netty所有的 I/O 操作都是异步。netty重新创造了一个继承jdk的Futrue的Futrue，这样扩展的方法不但加强了异步的处理可用性，也可与Promise做成观察者模式，监听感兴趣的事件，提交线程的使用效率。
​

### ChannelInitializer
&ensp;&ensp;&ensp;&ensp;用于在某个Channel注册到EventLoop后，对这个Channel执行一些初始化操作。ChannelInitializer虽然会在一开始会被注册到Channel相关的pipeline里，但是在初始化完成之后，ChannelInitializer会将自己从pipeline中移除，不会影响后续的操作。
 
### ChannelHandler
&ensp;&ensp;&ensp;&ensp;netty的channelHandler是channel处理器，基于netty的业务处理，不管多么复杂，都是由channelHandler来做的，可能涉及到多个channelHandler，channelHandler分为多种类型：encoder、decoder、业务处理等。
​

## 服务端的解析
### 一段服务端的示例代码
``` JAVA
public void start() throws Exception {
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workGroup = new NioEventLoopGroup();
    try { 
        _//create ServerBootstrap instance 
        _ServerBootstrap b = new ServerBootstrap(); 
        _//Specifies NIO transport, local socket address 
        //Adds handler to channel pipeline 
   _b.group(bossGroup,workGroup).channel(NioServerSocketChannel.class)`
                             `.localAddress(8080)
                .childHandler(new ChannelInitializer<Channel>() {
                    @Override 
                    protected void initChannel(Channel ch) throws Exception { 
                        ch.pipeline().addLast(new EchoServerHandler()); 
                    } 
                }); 
        _//Binds server, waits for server to close, and releases resources
        _ChannelFuture f = b.bind().sync(); 
        System._out_.println(EchoServer.class.getName() + "started and listen on " + f.channel().localAddress());
        f.channel().closeFuture().sync(); 
    } finally { 
        group.shutdownGracefully().sync(); 
    } 
}
```
​

&ensp;&ensp;&ensp;&ensp;其中关键的启动过程为：

- new NioEventLoopGroup中在创建没一个NioEventLoop时会通过SelectorProvider.openSelector创建对应的selector
- 创建channel工厂，利用泛型+反射+工厂来创建NioServerSocketChannel对象，其中包含了
   - 创建NioMessageUnsafe，用于netty底层的读写操作
   - 创建ChannelPipeline，默认的是DefaultChannelPipeline
- 然后向parentGroup的selector注册NioServerSocketChannel。
- 后续的话会轮询NioEvenLoop的run方法，根据里面的selector.select()来执行发生的事件处理。

​

​

## 客户端的解析
### 一段客户端的示例代码
``` JAVA
public void start() throws Exception { 
    EventLoopGroup group = new NioEventLoopGroup(); 
    try { 
        Bootstrap b = new Bootstrap(); 
        b.group(group).channel(NioSocketChannel.class).remoteAddress(new InetSocketAddress("127.0.0.1", 8080)) 
                .handler(new ChannelInitializer<SocketChannel>() { 
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception { 
                        ch.pipeline().addLast(new EchoClientHandler()); 
                    } 
                }); 
        ChannelFuture f = b.connect().sync(); 
        f.channel().closeFuture().sync(); 
    } finally { 
        group.shutdownGracefully().sync(); 
    } 
}
```
​

&ensp;&ensp;&ensp;&ensp;其中关键的启动过程为：

- new NioEventLoopGroup中在创建没一个NioEventLoop时会通过SelectorProvider.openSelector创建对应的selector
- 创建channel工厂，利用泛型+反射+工厂来创建NioSocketChannel对象
- 然后向group的selector注册NioSocketChannel。



## 参数优化

- WRITE_BUFFER_WATER_MARK:bufferbuf高低水位线，间接防止写数据OOM，默认 32k -> 64k。
- CONNECT_TIMEOUT_MILLIS:客户端连接服务器最大允许时间，默认30秒，建议设置为10秒。
