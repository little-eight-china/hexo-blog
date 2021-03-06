---
title: 'java线程池'
date: 2021-09-02 21:56:11
categories: 
  - java
tags: 
  - little_eight
---

# 线程池是什么
&ensp;&ensp;&ensp;&ensp;线程池是一种基于池化思想管理线程的工具,是存储线程的容器,线程事先创建好后放入线程池,当有任务需要执行时,直接从线程池拿空闲线程使用,使用完毕后归还给线程池.
<!--more-->
# 为什么使用线程池
&ensp;&ensp;&ensp;&ensp;线程过多会带来额外的开销，其中包括创建销毁线程的开销、调度线程的开销等等，同时也降低了计算机的整体性能。
&ensp;&ensp;&ensp;&ensp;线程池维护多个线程，等待监督管理者分配可并发执行的任务。
&ensp;&ensp;&ensp;&ensp;这种做法，一方面避免了处理任务时创建销毁线程开销的代价，另一方面避免了线程数量膨胀导致的过分调度问题，保证了对内核的充分利用。
&ensp;&ensp;&ensp;&ensp;可以用下面几点来概括使用线程池的好处。

* 降低资源消耗：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
* 提高响应速度：任务到达时，无需等待线程创建即可立即执行。
* 提高线程的可管理性：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
* 提供更多更强大的功能：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。

# 线程池核心设计与实现

## ThreadPoolExecutor类

### 核心类关系
&ensp;&ensp;&ensp;&ensp;ThreadPoolExecutor -> AbstractExecutorService -> ExecutorService -> Executor

&ensp;&ensp;&ensp;&ensp;ThreadPoolExecutor实现的顶层接口是Executor，顶层接口Executor提供了一种思想：将任务提交和任务执行进行解耦。用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由Executor框架完成线程的调配和任务的执行部分。ExecutorService接口增加了一些能力：（1）扩充执行任务的能力，补充可以为一个或一批异步任务生成Future的方法；（2）提供了管控线程池的方法，比如停止线程池的运行。

&ensp;&ensp;&ensp;&ensp;AbstractExecutorService则是上层的抽象类，将执行任务的流程串联了起来，保证下层的实现只需关注一个执行任务的方法即可。最下层的实现类ThreadPoolExecutor实现最复杂的运行部分，ThreadPoolExecutor将会一方面维护自身的生命周期，另一方面同时管理线程和任务，使两者良好的结合从而执行并行任务。

&ensp;&ensp;&ensp;&ensp;ThreadPoolExecutor是如何运行，如何同时维护线程和执行任务的呢？其运行机制如下:

### 七大参数
* corePollSize：核心线程数。在创建了线程池后，线程中没有任何线程，等到有任务到来时才创建线程去执行任务。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中。
* maximumPoolSize：最大线程数。表明线程中最多能够创建的线程数量。
* keepAliveTime：空闲的线程保留的时间。
* TimeUnit：空闲线程的保留时间单位。
* BlockingQueue：阻塞队列，存储等待执行的任务。
* ThreadFactory：线程工厂，用来创建线程
* RejectedExecutionHandler：队列已满，而且任务量大于最大线程的异常处理策略。

### 四大默认线程池
* newCachedThreadPool：创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。容易造成堆外内存溢出OOM，因为它的最大值是在初始化的时候设置为 Integer.MAX_VALUE
* newFixedThreadPool: 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
* newScheduledThreadPool: 创建一个定长线程池，支持定时及周期性任务执行。
* newSingleThreadExecutor: 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

### 生命周期管理
&ensp;&ensp;&ensp;&ensp;线程池运行的状态，并不是用户显式设置的，而是伴随着线程池的运行，由内部来维护。线程池内部使用一个变量维护两个值：运行状态(runState)和线程数量 (workerCount)。在具体实现中，线程池将运行状态(runState)、线程数量 (workerCount)两个关键参数的维护放在了一起，如下代码所示：
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
&ensp;&ensp;&ensp;&ensp;ctl这个AtomicInteger类型，是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段， 它同时包含两部分的信息：线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，高3位保存runState，低29位保存workerCount，两个变量之间互不干扰。用一个变量去存储两个值，可避免在做相关决策时，出现不一致的情况，不必为了维护两者的一致，而占用锁资源。通过阅读线程池源代码也可以发现，经常出现要同时判断线程池运行状态和线程数量的情况。线程池也提供了若干方法去供用户获得线程池当前的运行状态、线程个数。这里都使用的是位运算的方式，相比于基本运算，速度也会快很多。

&ensp;&ensp;&ensp;&ensp;关于内部封装的获取生命周期状态、获取线程池线程数量的计算方法如以下代码所示：
```java
private static int runStateOf(int c)     { return c & ~CAPACITY; } //计算当前运行状态
private static int workerCountOf(int c)  { return c & CAPACITY; }  //计算当前线程数量
private static int ctlOf(int rs, int wc) { return rs | wc; }   //通过状态和线程数生成ctl
```

&ensp;&ensp;&ensp;&ensp;ThreadPoolExecutor的运行状态有5种，分别为：
* RUNNING：能接受新提交的任务，也能处理阻塞队列中的任务。
* SHUTDOWN: 不再接受新提交的任务，但却可以继续处理阻塞队列的中的任务。
* STOP: 不能接受新的任务，也不处理队列中的任务，同时终止正在处理任务的线程。
* TIDYING: 所有任务都已终止了，workerCount有效线程数为0。
* TERMINATEN: 在进行terminaten()方法后进入此状态。

###  任务执行机制

&ensp;&ensp;&ensp;&ensp;任务调度是线程池的主要入口，当用户提交了一个任务，接下来这个任务将如何执行都是由这个阶段决定的。了解这部分就相当于了解了线程池的核心运行机制。

&ensp;&ensp;&ensp;&ensp;首先，所有任务的调度都是由execute方法完成的，这部分完成的工作是：检查现在线程池的运行状态、运行线程数、运行策略，决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。其执行过程如下：

&ensp;&ensp;&ensp;&ensp;首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
- 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
- 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
- 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
- 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。
其执行流程如下图所示：

### 任务缓冲
&ensp;&ensp;&ensp;&ensp;任务缓冲模块是线程池能够管理任务的核心部分。线程池的本质是对任务和线程的管理，而做到这一点最关键的思想就是将任务和线程两者解耦，不让两者直接关联，才可以做后续的分配工作。线程池中是以生产者消费者模式，通过一个阻塞队列来实现的。阻塞队列缓存任务，工作线程从阻塞队列中获取任务。

&ensp;&ensp;&ensp;&ensp;阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

&ensp;&ensp;&ensp;&ensp;使用不同的队列可以实现不一样的任务存取策略。在这里，我们可以再介绍下阻塞队列的成员：

* ArrayBlockingQueue: 一个用数组实现的有界阻塞队列。此队列按照先进先出的原则对元素进行排序。支持公平锁和非公平锁。
* LinkedBlockingQueue：一个由链表结构组成的有界队列。此队列按照先进先出的原则对元素进行排序。默认长度是Integer最大值。
* PriorityBlockingQueue：一个支持线程优先级排序的无界队列，默认自然排序，也可以自定义实现排序规则。
* DelayQueue：一个实现实现延迟获取的无界队列。在创建元素的时候，可以指定多久才能从队列中获取当前元素。只有延时期满后才能从队列中获取元素。
* SynchronousQueue：一个不存储元素的阻塞队列，每一个put操作必须等待take操作，否则不添加新元素。
* LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
* LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。队列头部跟尾部都可以添加和移除元素，多线程并发时，可以将锁的竞争最多降到一半、

### 任务申请
&ensp;&ensp;&ensp;&ensp;任务的执行有两种可能：一种是任务直接由新创建的线程执行。另一种是线程从任务队列中获取任务然后执行，执行完任务的空闲线程会再次去从队列中申请任务再去执行。第一种情况仅出现在线程初始创建的时候，第二种是线程获取任务绝大多数的情况。


&ensp;&ensp;&ensp;&ensp;getTask这部分进行了多次判断，为的是控制线程的数量，使其符合线程池的状态。如果线程池现在不应该持有那么多线程，则会返回null值。工作线程Worker会不断接收新任务去执行，而当工作线程Worker接收不到任务的时候，就会开始被回收。

### 任务拒绝

&ensp;&ensp;&ensp;&ensp;任务拒绝模块是线程池的保护部分，线程池有一个最大的容量，当线程池的任务缓存队列已满，并且线程池中的线程数目达到maximumPoolSize时，就需要拒绝掉该任务，采取任务拒绝策略，保护线程池。



