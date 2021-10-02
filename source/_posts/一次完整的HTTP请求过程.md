---
title: '一次完整的HTTP请求过程'
date: 2021-09-02 00:15:20
categories: 
  - 网络
tags: 
  - little_eight
---
## 域名解析
&ensp;&ensp;&ensp;&ensp;搜索浏览器自身的dns缓存，找不到就搜索系统的缓存，还是找不到就去hosts里找，还是找不到就去域名服务器里找。搜索成功拿到ip的话，接下来就是tcp连接。
​<!--more-->

## tcp三次握手

1. 第一次握手：Client将标志位SYN置为1，以及初始序号seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。
1. 第二次握手：Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack=J+1，以及初始序号seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。
1. 第三次握手：Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以开始传输数据了。
## http请求解析
&ensp;&ensp;&ensp;&ensp;建立TCP连接之后，Web浏览器会向Web服务器发送请求命令。例如：GET /sample/hello.jsp HTTP/1.1。浏览器发送请求信息之后，还要以头信息的形式发送相关信息，并以空行代表发送结束。
&ensp;&ensp;&ensp;&ensp;HTTP请求报文由3部分组成（**请求行+请求头+请求体**）
​

![](https://gitee.com/littleeight/blog-images/raw/master/%E4%B8%80%E6%AC%A1%E5%AE%8C%E6%95%B4%E7%9A%84http%E8%AF%B7%E6%B1%82%E8%BF%87%E7%A8%8B/1.png)

以tomcat为例，tomcat对外服务的部分是连接器，由连接器进行解析和处理交给catalina容器。
![](https://gitee.com/littleeight/blog-images/raw/master/%E4%B8%80%E6%AC%A1%E5%AE%8C%E6%95%B4%E7%9A%84http%E8%AF%B7%E6%B1%82%E8%BF%87%E7%A8%8B/2.png)


&ensp;&ensp;&ensp;&ensp;由于协议不同，客户端发过来的请求信息也不尽相同，Tomcat定义了⾃⼰的 Request类来封装这些请求信息，ProtocolHandler接⼝负责解析请求并⽣成 Tomcat Request类，
&ensp;&ensp;&ensp;&ensp;通过适配器CoyoteAdapter调⽤Sevice⽅法，传⼊的是Tomcat Request对象， CoyoteAdapter负责将Tomcat Request转成ServletRequest
，在Servlet容器找到对应可以处理请求的Servlet。
&ensp;&ensp;&ensp;&ensp;假如说是Spring的DispatchServlet，它会去HandlerMapping处理器映射器找到处理器适配器HandlerAdapter
&ensp;&ensp;&ensp;&ensp;找到真正处理的方法Handler，进行后台逻辑处理，HandlerAdapter将执行结果ModelAndView返回给DispatcherServlet，DispatcherServlet将ModelAndView传给ViewReslover视图解析器，ViewReslover解析后返回具体View，DispatcherServlet根据View进行渲染视图（即将模型数据填充至视图中），最终DispatcherServlet响应用户。
​

&ensp;&ensp;&ensp;&ensp;服务器向客户机回送应答， HTTP/1.1 200 OK ，应答的第一部分是协议的版本号和应答状态码。
&ensp;&ensp;&ensp;&ensp;随同应答向用户发送关于它自己的数据及被请求的文档。
&ensp;&ensp;&ensp;&ensp;Web服务器向浏览器发送头信息后，它会发送一个空白行来表示头信息的发送到此为结束，接着，它就以Content-Type应答头信息所描述的格式发送用户所请求的实际数据。
​

HTTP的响应报文：
​

![](https://gitee.com/littleeight/blog-images/raw/master/%E4%B8%80%E6%AC%A1%E5%AE%8C%E6%95%B4%E7%9A%84http%E8%AF%B7%E6%B1%82%E8%BF%87%E7%A8%8B/3.png)





## tcp四次挥手


1. 第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
1. 
第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。此时server还可以继续发送未发完的数据。
1. 第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
1. 第四次挥手：Client收到FIN后，接着发送一个ACK给Server，然后Client进入TIME_WAIT状态（看下面的tcp状态），如果server未收到则会重发fin包，然后client接收到之后重复步骤4。Server确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。

​

![](https://gitee.com/littleeight/blog-images/raw/master/%E4%B8%80%E6%AC%A1%E5%AE%8C%E6%95%B4%E7%9A%84http%E8%AF%B7%E6%B1%82%E8%BF%87%E7%A8%8B/4.png)




## 请求过程的一些优化点
### ​

&ensp;&ensp;&ensp;&ensp;首先** dns** 这块能不能进行一个优化呢？我们可以在这层做一个浏览器缓存。这样的话，返回 dns 的时间就可以缩短很多。
&ensp;&ensp;&ensp;&ensp;然后就是一个网络请求的过程，网络请求的过程涉及到带宽，涉及到网络的选择，涉及到缓存，那么在这个过程中，我们有没有什么优化的点，实际在网络请求的过程中，很多公司基本上都会使用到** CDN** , 使用了 CDN 就实际上解决了网络选择以及缓存的问题。但是在访问 CDN 的过程中还会涉及到一个问题。就是 CDN 是请求静态资源用的。那么对于静态资源来说，实际上，我们请求所带的 cookie 是没有用的。实际上在请求 CDN 的过程中，能把 cookie 给去掉。但是很多时候我们 CDN 域名会弄得跟网站本身的域名相同。那么就会将主站的一些 cookie 。通过我们的网络携带到 CDN 的服务端。这个实际上是对网络无畏的损耗，所以我们的 CDN 的域名要非常注意，这个 CDN 的域名不要和主站的一样。这样的话就可以放置访问 CDN 的时候还携带主站 cookie 这个问题，使用 CDN 可以解决我们静态资源，网络选择，以及缓存的问题。但是对于一些接口是没法使用 CDN 的。实际上可以在浏览器上做缓存
&ensp;&ensp;&ensp;&ensp;除了缓存，路径选择，带宽也是非常重要的一点。
&ensp;&ensp;&ensp;&ensp;如果一个** **http 请求的大小会小一点，那么返回的速度相对会快一些。所以**如何减少 http 请求的大小**也是整个优化中非常重要的一点。
&ensp;&ensp;&ensp;&ensp;另外每一个 http 请求都会经过网络环境到达服务器。实际上每次请求都有一个网络环境的损耗。那么能否**将多次 http 请求合并成一次**，从而减少相同的网络损耗呢
&ensp;&ensp;&ensp;&ensp;最后就是浏览器端的渲染过程。对于现在的一些大型框架来说，比如 react,vue 来说。这样的框架，他的模板都是在浏览器端进行渲染的，不是直出的 html ，而是要通过相应的框架代码才能渲染出我们的页面。这个渲染过程，对于首屏就有很大的损耗。这个其实是不利于前端的性能的。在这个情况下，业界就会有一个服务端渲染的方案，在服务端进行这个 html 渲染，从而将这个 html 直出到浏览器端。而不是在浏览器端进行渲染，所以渲染这层，可以进行服务端渲染以及渲染的方案。
​

&ensp;&ensp;&ensp;&ensp;设置tomcat的最大线程数，以及配合集群可达到较高的并发量。
