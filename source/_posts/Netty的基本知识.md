---
title: 'Netty的基本知识'
date: 2021-07-02 17:22:00
categories: 
  - java第三方框架
tags: 
  - little_eight
---
## 基本介绍
Netty 是一个 NIO 客户机服务器框架，它支持快速简单地开发网络应用程序，如协议服务器和客户机。它极大地简化了网络编程，如 TCP 和 UDP 套接字服务器。经过多年的发展它已成为构建一个java网络生态的首选架构，在一些开源的项目中可见。
下面是netty框架的组成：
![](https://gitee.com/littleeight/blog-images/raw/master/netty%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%9F%A5%E8%AF%86/1.png)

Netty是一个非阻塞、事件驱动的网络框架，整个Netty的api都是异步设计，对网络应用来讲，io一般是性能的瓶颈，使用异步io可以较大程度上提高程序性能，对于熟悉多线程编程的读者可能会需要同步代码。这样的方式不好，因为同步会影响程序的性能，Netty的设计保证程序处理事件不会有同步。
<!--more--> 

​

## 核心类
​

一个Netty程序开始于Bootstrap类，Bootstrap类是Netty提供的一个可以通过简单配置来设置或"引导"程序的一个很重要的类。Netty中设计了Handlers来处理特定的"event"和设置Netty中的事件，从而来处理多个协议和数据。事件可以描述成一个非常通用的方法，因为你可以自定义一个handler,用来将Object转成byte[]或将byte[]转成Object；也可以定义个handler处理抛出的异常。 
你会经常编写一个实现ChannelInboundHandler的类，ChannelInboundHandler是用来接收消息，当有消息过来时，你可以决定如何处理。当程序需要返回消息时可以在ChannelInboundHandler里write/flush数据。可以认为应用程序的业务逻辑都是在ChannelInboundHandler中来处理的，业务罗的生命周期在ChannelInboundHandler中。 
Netty连接客户端端或绑定服务器需要知道如何发送或接收消息，这是通过不同类型的handlers来做的，多个Handlers是怎么配置的？Netty提供了ChannelInitializer类用来配置Handlers。ChannelInitializer是通过ChannelPipeline来添加ChannelHandler的，如发送和接收消息，这些Handlers将确定发的是什么消息。ChannelInitializer自身也是一个ChannelHandler，在添加完其他的handlers之后会自动从ChannelPipeline中删除自己。 
所有的Netty程序都是基于ChannelPipeline。ChannelPipeline和EventLoop和EventLoopGroup密切相关，因为它们三个都和事件处理相关，所以这就是为什么它们处理IO的工作由EventLoop管理的原因。 
 	Netty中所有的IO操作都是异步执行的，例如你连接一个主机默认是异步完成的；写入/发送消息也是同样是异步。也就是说操作不会直接执行，而是会等一会执行，因为你不知道返回的操作结果是成功还是失败，但是需要有检查是否成功的方法或者是注册监听来通知；Netty使用Futures和ChannelFutures来达到这种目的。Future注册一个监听，当操作成功或失败时会通知。ChannelFuture封装的是一个操作的相关信息，操作被执行时会立刻返回ChannelFuture。


### 一段客户端的示例代码
`public void start() throws Exception { 
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
}`
​

​

### 一段服务端的示例代码
`public void start() throws Exception {
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
}`
### Bootstrap、ServerBootstrap
      相当于客户端与服务端。
Bootstrap和ServerBootstrap之间的差异： 
Bootstrap用来连接远程主机，有1个EventLoopGroup,专门处理连接。
ServerBootstrap用来绑定本地端口，有2个EventLoopGroup，一个处理客户端的新连接，一个处理与客户端的交互。前者会轮询将连接交给后者的一个NioEventLoop处理。
​

Netty的启动流程中，涉及到多个操作，比如register、bind、注册对应事件等，为了不影响main线程执行，这些工作以task的形式提交给NioEventLoop，由NioEventLoop来执行这些task，也就是register、bind、注册事件等操作。


​

### EventLoop、EventLoopGroup 
​

EventLoopGroup中可能包含了多个EventLoop，EventLoop是一个Reactor模型的事件处理器，一个EventLoop对应一个线程，其内部会维护一个selector和taskQueue（fifo的队列），负责处理客户端请求和内部任务，内部任务如ServerSocketChannel注册和ServerSocket绑定操作等。
IO事件和内部任务执行时间百分比通过ioRatio来调节，ioRatio表示执行IO时间所占百分比。任务包括普通任务和已经到时的延迟任务，延迟任务存放到一个优先级队列PriorityQueue中，执行任务前从PriorityQueue读取所有到时的task，然后添加到taskQueue中，最后统一执行task。
​

### Channel
在Nio网络编程模型中, 服务端和客户端进行IO数据交互(得到彼此推送的信息)的媒介就是Channel。Netty的channel包含了以下信息：
id
可能存在的parent Channel
管道 pepiline
用于数据读写的unsafe内部类
关联上相伴终生的NioEventLoop
​

Channel提供了很多方法，如下列表： 
eventLoop()，返回分配给Channel的EventLoop 
pipeline()，返回分配给Channel的ChannelPipeline 
isActive()，返回Channel是否激活，已激活说明与远程连接对等 
localAddress()，返回已绑定的本地SocketAddress 
remoteAddress()，返回已绑定的远程SocketAddress 
write()，写数据到远程客户端，数据通过ChannelPipeline传输过去


### ChannelPipeline 
​

每个channel内部都会持有一个ChannelPipeline对象pipeline，ChannelPipeline 提供了一个容器给 ChannelHandler 链并提供了一个API 用于管理沿着链入站和出站事件的流动。pipeline默认实现DefaultChannelPipeline内部维护了一个DefaultChannelHandlerContext链表。
channel的读写操作都会走到DefaultChannelPipeline中，当channel完成register、active、read、readComplete等操作时，会触发pipeline的相应方法。
当channel注册到selector后，触发pipeline的fireChannelRegistered方法；
当channel是可用时，触发pipeline的fireChannelActive方法。（fireChannelActive触发一般是在fireChannelRegistered之后触发的）；
当客户端发送数据时，触发pipeline的fireChannelRead方法；
触发pipeline的fireChannelRead方法之后会触发pipeline的fireChannelReadComplete方法。
​

### Future or ChannelFuture
​

Netty 所有的 I/O 操作都是异步。因为一个操作可能无法立即返回，我们需要有一种方法在以后获取它的结果。出于这个目的，Netty 提供了接口 ChannelFuture,它的 addListener 方法。
​

### ChannelInitializer
​

用于在某个Channel注册到EventLoop后，对这个Channel执行一些初始化操作。ChannelInitializer虽然会在一开始会被注册到Channel相关的pipeline里，但是在初始化完成之后，ChannelInitializer会将自己从pipeline中移除，不会影响后续的操作。
 
### ChannelHandler
netty的channelHandler是channel处理器，基于netty的业务处理，不管多么复杂，都是由channelHandler来做的，可能涉及到多个channelHandler，channelHandler分为多种类型：encoder、decoder、业务处理等。
​

## 粘包半包
粘包就是接收端读取时，多个发送过来的bytebuf粘到了一起，半包就是一个完整的bytebuf被分成了2部分2次发送。例如：
1、服务端一次接收到了两个数据包，D1和D2粘合在一起，被称为TCP粘包；
2、服务端分两次读取到了两个数据包，第一次读取到了完整的D1包和D2包的部分内容，3、第二次读取到了D2包的剩余内容，这被称为TCP拆包
4、服务端分两次读取到了两个数据包，第一次读取到了D1包的部分内容D1_1，第二次读取到了D1包的剩余内容D1_2和D2包的整包。
​

### 解决思路
基本思路是，在接收端，需要根据自定义协议来，来读取底层的数据包，重新组装我们应用层的数据包，这个过程通常在接收端称为**拆包**。
1、接收端应用层不断从底层的TCP 缓冲区中读取数据。
2、每次读取完，判断一下是否为一个完整的应用层数据包。如果是，上层应用层数据包读取完成。
3、如果不是，那就保留该数据在应用层缓冲区，然后继续从 TCP 缓冲区中读取，直到得到一个完整的应用层数据包为止。
4、至此，半包问题得以解决。
5、如果从TCP底层读到了多个应用层数据包，则将整个应用层缓冲区，拆成一个一个的独立的应用层数据包，返回给调用程序。
6、至此，粘包问题得以解决。
​

### 解决方案
Netty提供了4种解码器（基类都是ByteToMessageDecoder）来解决，分别如下：
1、固定长度的拆包器 FixedLengthFrameDecoder，每个应用层数据包的都拆分成都是固定长度的大小，通过在包头增加消息体长度的解码器，解析数据时首先获取首部长度，然后定长读取socket中的数据。
2、行拆包器 LineBasedFrameDecoder，每个应用层数据包，都以换行符作为分隔符，进行分割拆分。换行符解码器，报文尾部增加固定换行符rn，解析数据时以换行符作为报文结尾。
3、分隔符拆包器 DelimiterBasedFrameDecoder，每个应用层数据包，都通过自定义的分隔符，进行分割拆分。分隔符解码器，使用特定分隔符作为报文的结尾，解析数据时以定义的分隔符作为报文结尾
4、基于数据包长度的拆包器 LengthFieldBasedFrameDecoder，将应用层数据包的长度，作为接收端应用层数据包的拆分依据。按照应用层数据包的大小，拆包。这个拆包器，有一个要求，就是应用层协议中包含数据包的长度。定长解码器，这个最简单，消息体固定长度，解析数据时按长度读取即可
