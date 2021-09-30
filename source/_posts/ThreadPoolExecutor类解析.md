---
title: ThreadPoolExecutor类解析
date: 2019-07-23 20:07:44
categories: 
  - java
tags: 
  - little_eight
---

  在《阿里巴巴java开发手册》中指出了线程资源必须通过线程池提供，不允许在应用中自行显示的创建线程，这样一方面是线程的创建更加规范，可以合理控制开辟线程的数量；另一方面线程的细节管理交给线程池处理，优化了资源的开销。而线程池不允许使用Executors去创建，而要通过ThreadPoolExecutor方式，这一方面是由于jdk中Executor框架虽然提供了如newFixedThreadPool()、newSingleThreadExecutor()、newCachedThreadPool()等创建线程池的方法，但都有其局限性，不够灵活；另外由于前面几种方法内部也是通过ThreadPoolExecutor方式实现，使用ThreadPoolExecutor有助于大家明确线程池的运行规则，创建符合自己的业务场景需要的线程池，避免资源耗尽的风险。
##  成员变量
```
    /**
     * 线程池的控制状态，是AtomicInteger类型的，里面包含两部分，workcount---线程的数量，
     * runState---线程池的运行状态。这里限制了最大线程数是2^29-1，大约500百万个线程，
     * 这也是个问题，所以ctl也可以变成AtomicLong类型的
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    /**
     * 线程数量所占位数
     */
    private static final int COUNT_BITS = Integer.SIZE - 3;
    /**
     * 理论上的最大活跃线程数
     */
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    /**
     * RUNNING - 接受新任务并且继续处理阻塞队列中的任务
     * SHUTDOWN - 不接受新任务但是会继续处理阻塞队列中的任务
     * STOP -  不接受新任务，不在执行阻塞队列中的任务，中断正在执行的任务
     * TIDYING - 所有任务都已经完成，线程数都被回收，线程会转到TIDYING状态会继续执行钩子方法
     * TERMINATED - 钩子方法执行完毕
     */
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
    
    
    /**
     *  存放任务的队列，只有当线程数>核心线程数，才会把其他的任务放入queue，
     * 一般常用的是queue就是ArrayBlockingQueue，LinkedBlockingQueue，
     * SynchronousQueue， ConcurrentLinkedQueue。
     *
　　 * 1）ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；
     *    
　　 * 2）LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；
　　 * 3）synchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。
　　 * 4) ConcurrentLinkedQueue: 无界线程安全队列
     */
    private final BlockingQueue<Runnable> workQueue;

    /**
     * 包含池中所有工作线程的集合。仅当保持主锁.
     */
    private final HashSet<Worker> workers = new HashSet<Worker>();

    /**
     * 支持等待终止的等待条件
     */
    private final Condition termination = mainLock.newCondition();

    /**
     * 跟踪获得的最大池大小。仅在主锁下访问。
     */
    private int largestPoolSize;

    /**
     * 已完成任务的计数器。仅在工作线程终止时更新。仅在主锁下访问
     */
    private long completedTaskCount;

    /**
     * 创建线程的工厂类
     */
    private volatile ThreadFactory threadFactory;

    /**
     * 在执行中饱和或关闭时调用的处理程序。拒绝策略；当任务太多来不及处理时，如何拒绝任务
     */
    private volatile RejectedExecutionHandler handler;

    /**
     * 当线程池中创建的线程超过了核心线程数的时候，这些多余的空闲线程在结束之前等待新的                  * 创建线程的工厂类任务最大的存活时间。
     */
    private volatile long keepAliveTime;

    /**
     * 允许核心线程被回收
     */
    private volatile boolean allowCoreThreadTimeOut;

    /**
     * 线程池中的核心线程数，空闲的线程也不会回收，除非把allowCoreThreadTimeOut设置为true，              * 这时核心线程才会被回收
     */
    private volatile int corePoolSize;

    /**
     * 线程池中可以创建的最大线程数，限定为2^29-1，大约500百万个线程。
     * 需要注意的是，当使用无界的阻塞队列的时候，maximumPoolSize就起不到作用了。
     */
    private volatile int maximumPoolSize;

    /**
     * 默认被拒绝的执行处理程序
     */
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();

    /**
     * 终止的权限
     */
    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");

    /**
     * 
     */
    private final AccessControlContext acc;
```
<!--more-->
##  构造方法
```
    /**
     * 有多个构造方法，只讲最终实现的
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        // 设置的核心线程数小于0，或者最大线程数小于0，或者最大线程数小于核心线程数，
        // 创建线程的工厂类任务最大的存活时间小于0都抛出异常
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        // 存放任务的队列、线程工厂、处理程序为null也抛出异常
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        // 下面就是设置参数了
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
    
```

## 主要方法（从用法入手）

### execute
```
   /**
    * 执行入口
    *
    * 三步操作
    *
    * 1. 如果当前运行的线程数<核心线程数,创建一个新的线程执行任务,调用addWorker方法原子性地检查
    *    运行状态和线程数,通过返回false防止不需要的时候添加线程
    * 2. 如果一个任务能够成功的入队,仍然需要双重检查,因为我们添加了一个线程(有可能这个线程在上次检查后就已经死亡了)
    *    或者进入此方法的时候调用了shutdown,所以需要重新检查线程池的状态,如果必要的话,当停止的时候要回滚入队操作,
    *    或者当线程池为空的话创建一个新的线程
    * 3. 如果不能入队,尝试着开启一个新的线程,如果开启失败,说明线程池已经是shutdown状态或饱和了,所以拒绝执行该任务
    */
     public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        // 获取当前线程池的控制状态
        int c = ctl.get();
        // 如果当前运行的线程数<核心线程数
        if (workerCountOf(c) < corePoolSize) {
            // 调用addWorker方法原子性地检查运行状态和线程数,通过返回false防止不需要的时候添加线程
            // 添加worker,成功则返回,下面再解析这个方法
            if (addWorker(command, true))
                return;
            // 不成功则再次获取线程池控制状态
            c = ctl.get();
        }
        // 线程池处于RUNNING状态，将命令（用户自定义的Runnable对象）添加进workQueue队列
        if (isRunning(c) && workQueue.offer(command)) {
             // 再次检查，获取线程池控制状态
            int recheck = ctl.get();
            // 线程池不处于RUNNING状态，将命令从workQueue队列中移除
            if (! isRunning(recheck) && remove(command))
                // 拒绝执行命令
                reject(command);
            // worker数量等于0,添加worker
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 添加worker失败,拒绝执行命令
        else if (!addWorker(command, false))
            reject(command);
    }
```
### addWorker
```
    /**
     * 
     */
     private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 检查线程池状态是否正常，否则返回false
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                // 检查工作线程数是否正常，否则返回false
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 用到了原子CAS方法比较，使用CAS增加worker计数器成功，才能进入下一步
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get(); 
                 // 这里表示执行到这里的时候线程池的运行状态发生改变的话，需要重新跳到retry处执行
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }
        // worker开始标识
        boolean workerStarted = false;
        // worker被添加标识
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 使用firstTask初始化Worker，first可能为null，那么则表示该worker为空闲
            w = new Worker(firstTask);
            // 获取worker对应的线程
            final Thread t = w.thread;
            if (t != null) {
                 // 获取线程池锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // 线程池的运行状态
                    int rs = runStateOf(ctl.get());
                    // 判断线程池状态
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 线程刚添加进来，还未启动就存活,抛出线程状态异常
                        if (t.isAlive())
                            throw new IllegalThreadStateException();
                        // 添加worker
                        workers.add(w);
                        int s = workers.size();
                        // 如果队列大小大于最大池大小，让后者等于前者
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        // 标识worker添加成功
                        workerAdded = true;
                    }
                } finally {
                    // 解锁
                    mainLock.unlock();
                }
                // 如果worker添加成功，就标识运行成功
                if (workerAdded) {
                    // 开始执行worker的run方法
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // worker没有运行
            if (! workerStarted)
                // 添加worker失败，后面解析这个方法
                addWorkerFailed(w);
        }
        // 返回worker的运行状态
        return workerStarted;
    }
```

### addWorkerFailed
```
    /**
     * 
     */
     private void addWorkerFailed(Worker w) {
        // 获取主锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                // 移除该worker
                workers.remove(w);
            // 数量减1
            decrementWorkerCount();
            // 调用tryTerminate方法来尝试中止线程池,或者是清理一下线程池，下面说
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```

### tryTerminate
```
    /**
     * 尝试终止线程池
     */
final void tryTerminate() {
        for (;;) {
            // 获取线程池控制状态
            int c = ctl.get();
            // 线程池的运行状态为RUNNING
            if (isRunning(c) ||      
                // 线程池的运行状态最小要大于TIDYING
                runStateAtLeast(c, TIDYING) ||   
                 // 线程池的运行状态为SHUTDOWN并且workQueue队列不为null
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))   
                // 不能终止，直接返回
                return;
            // 线程池正在运行的worker数量不为0   
            if (workerCountOf(c) != 0) { 
                // 仅仅中断一个空闲的worker，下面说
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            // 获取线程池的锁
            final ReentrantLock mainLock = this.mainLock;
            // 获取锁
            mainLock.lock();
            try {
                // 比较并设置线程池控制状态为TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { 
                    try {
                        // 终止，钩子函数
                        terminated();
                    } finally {
                        // 设置线程池控制状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        // 释放在termination条件上等待的所有线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

### interruptIdleWorkers

```
    /**
     * 尝试中断线程，onlyOne标识是否只中断一个
     */
private void interruptIdleWorkers(boolean onlyOne) {
        // 线程池的锁
        final ReentrantLock mainLock = this.mainLock;
        // 获取锁
        mainLock.lock();
        try {
            for (Worker w : workers) { // 遍历workers队列
                // worker对应的线程
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) { // 线程未被中断并且成功获得锁
                    try {
                        // 中断线程
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        // 释放锁
                        w.unlock();
                    }
                }
                if (onlyOne) // 若只中断一个，则跳出循环
                    break;
            }
        } finally {
            // 释放锁
            mainLock.unlock();
        }
    }
```


### runWorker 重点关注这个
```
    /**
     * 
     */
     final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
             // 不断循环getTask来获取任务，getTask后面说
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 如果当前线程是stop，那么将确认其为interrupted
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 调用钩子函数
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 调用钩子函数
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 处理完成后，调用,下面会说
            processWorkerExit(w, completedAbruptly);
        }
    }
```

### getTask
```
    /**
     * 获取任务
     */
     private Runnable getTask() {
        // 超时标识
        boolean timedOut = false;

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 检验线程池状态 
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // 是否允许coreThread超时或者workerCount大于核心大小
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 检查线程数量
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    // 等待指定时间
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    // 一直等待，直到有元素
                    workQueue.take();
                if (r != null)
                    return r;
                // 等待指定时间后，没有获取元素，则超时
                timedOut = true;
            } catch (InterruptedException retry) {
                // 抛出了被中断异常，重试，没有超时
                timedOut = false;
            }
        }
    }
```


### processWorkerExit
```
    /**
     * 根据是否中断了空闲线程来确定是否减少workerCount的值，并且将worker从workers集合中移除并且会尝试终止线程池。
     */
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 如果被中断，则需要减少workCount
        if (completedAbruptly)
            decrementWorkerCount();
        // 获取可重入锁
        final ReentrantLock mainLock = this.mainLock;
        // 获取锁
        mainLock.lock();
        try {
            // 将worker完成的任务添加到总的完成任务中
            completedTaskCount += w.completedTasks;
            // 从workers集合中移除该worker
            workers.remove(w);
        } finally {
            // 释放锁
            mainLock.unlock();
        }
        // 尝试终止
        tryTerminate();
        // 获取线程池控制状态
        int c = ctl.get();
        // 小于STOP的运行状态
        if (runStateLessThan(c, STOP)) { 
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                 // 允许核心超时并且workQueue阻塞队列不为空
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // workerCount大于等于min
                if (workerCountOf(c) >= min) 
                    // 直接返回
                    return; // replacement not needed
            }
            // 添加worker
            addWorker(null, false);
        }
    }
```

