---
title: Java多线程
date: 2018-12-10 13:49:27
categories: 
  - java
tags: 
  - little_eight
---

## 线程基础
### 什么是线程？ 
&ensp;&ensp;&ensp;&ensp;几乎每种操作系统都支持进程的概念 —— 进程就是在某种程度上相互隔离的、独立运行的程序。

&ensp;&ensp;&ensp;&ensp;线程化是允许多个活动共存于一个进程中的工具。大多数现代的操作系统都支持线程，而且线程的概念以各种形式已存在了好多年。Java 是第一个在语言本身中显式地包含线程的主流编程语言，它没有把线程化看作是底层操作系统的工具。 

&ensp;&ensp;&ensp;&ensp;有时候，线程也称作轻量级进程。就象进程一样，线程在程序中是独立的、并发的执行路径，每个线程有它自己的堆栈、自己的程序计数器和自己的局部变量。但是，与分隔的进程相比，进程中的线程之间的隔离程度要小。它们共享内存、文件句柄和其它每个进程应有的状态。 

&ensp;&ensp;&ensp;&ensp;进程可以支持多个线程，它们看似同时执行，但互相之间并不同步。一个进程中的多个线程共享相同的内存地址空间，这就意味着它们可以访问相同的变量和对象，而且它们从同一堆中分配对象。尽管这让线程之间共享信息变得更容易，但您必须小心，确保它们不会妨碍同一进程里的其它线程。

&ensp;&ensp;&ensp;&ensp;Java 线程工具和 API 看似简单。但是，编写有效使用线程的复杂程序并不十分容易。因为有多个线程共存在相同的内存空间中并共享相同的变量，所以您必须小心，确保您的线程不会互相干扰。 

### 每个 Java 程序都使用线程

&ensp;&ensp;&ensp;&ensp;每个 Java 程序都至少有一个线程 — 主线程。当一个 Java 程序启动时，JVM 会创建主线程，并在该线程中调用程序的 main() 方法。 JVM 还创建了其它线程，您通常都看不到它们 — 例如，与垃圾收集、对象终止和其它 JVM 内务处理任务相关的线程。其它工具也创建线程，如 AWT（抽象窗口工具箱（Abstract Windowing Toolkit））或 Swing UI 工具箱、servlet 容器、应用程序服务器和 RMI（远程方法调用（Remote Method Invocation）） 

###  Java 线程的风险

&ensp;&ensp;&ensp;&ensp;虽然 Java 线程工具非常易于使用，但当我们创建多线程程序时，应该尽量避免一些风险。 
&ensp;&ensp;&ensp;&ensp;当多个线程访问同一数据项（如静态字段、可全局访问对象的实例字段或共享集合）时，需要确保它们协调了对数据的访问，这样它们都可以看到数据的一致视图，而且相互不会干扰另一方的更改。
&ensp;&ensp;&ensp;&ensp;为了实现这个目的，Java 语言提供了两个关键字：synchronized 和 volatile。当从多个线程中访问变量时，必须确保对该访问正确地进行了同步。对于简单变量，将变量声明成volatile 也许就足够了，但在大多数情况下，需要使用同步。 
&ensp;&ensp;&ensp;&ensp;如果将要使用同步来保护对共享变量的访问，那么必须确保在程序中所有访问该变量的地方都使用同步。

&ensp;&ensp;&ensp;&ensp;虽然线程可以大大简化许多类型的应用程序，过度使用线程可能会危及程序的性能及其可维护性。
&ensp;&ensp;&ensp;&ensp;线程消耗了资源。因此，在不降低性能的情况下，可以创建的线程的数量是有限制的。 尤其在单处理器系统中，使用多个线程不会使主要消耗 CPU 资源的程序运行得更快。 
<!--more-->
### 例子

&ensp;&ensp;&ensp;&ensp;以下示例使用两个线程，一个用于计时，一个用于执行实际工作。主线程使用非常简单的算法计算素数。 
&ensp;&ensp;&ensp;&ensp;在它启动之前，它创建并启动一个计时器线程，这个线程会休眠十秒钟，然后设置一个主线程要检查的标志。十秒钟之后，主线程将停止。请注意，共享标志被声明成 volatile。 

```java
public class CalculatePrimes extends Thread { 
    public static final int MAX_PRIMES = 1000000; 
    public static final int TEN_SECONDS = 10000; 
    public volatile boolean finished = false; 
    public void run() { 
        int[] primes = new int[MAX_PRIMES]; 
        int count = 0; 
        for (int i=2; count<MAX_PRIMES; i++) { 
        // Check to see if the timer has expired 
            if (finished) { 
            break; } 
        boolean prime = true; 
        for (int j=0; j<count; j++) { 
            if (i % primes[j] == 0) { 
                  prime = false; 
               break; 
            } 
        } 
        if (prime) { 
            primes[count++] = i; 
            System.out.println("Found prime: " + i); 
        } 
    } 
 } 
 public static void main(String[] args) { 
    CalculatePrimes calculator = new CalculatePrimes(); 
    calculator.start(); 
    try { 
    Thread.sleep(TEN_SECONDS); 
    } 
    catch (InterruptedException e) { 
    // fall through 
    } 
    calculator.finished = true; 
    } 
} 
```

## 线程的生命

### 创建线程

&ensp;&ensp;&ensp;&ensp;在 Java 程序中创建线程有几种方法。每个 Java程序至少包含一个线程：主线程。其它线程都是通过 Thread 构造器或实例化继承类 Thread 的类来创建的。

&ensp;&ensp;&ensp;&ensp;Java 线程可以通过直接实例化 Thread 对象或实例化继承 Thread 的对象来创建其它线程。在线程基础中的示例（其中，我们在十秒钟之内计算尽量多的素数）中，我们通过实例化 CalculatePrimes 类型的对象（它继承了 Thread），创建了一个线程。 

&ensp;&ensp;&ensp;&ensp;当我们讨论 Java 程序中的线程时，也许会提到两个相关实体：完成工作的实际线程或代表线程的Thread 对象。正在运行的线程通常是由操作系统创建的；Thread 对象是由 Java VM 创建的，作为控制相关线程的一种方式。 

### 创建线程和启动线程并不相同

&ensp;&ensp;&ensp;&ensp;在一个线程对新线程的 Thread 对象调用 start() 方法之前，这个新线程并没有真正开始执行。
&ensp;&ensp;&ensp;&ensp;Thread 对象在其线程真正启动之前就已经存在了，而且其线程退出之后仍然存在。这可以让您控制或获取关于已创建的线程的信息，即使线程还没有启动或已经完成了。 

&ensp;&ensp;&ensp;&ensp;通常在构造器中通过 start()启动线程并不是好主意。这样做，会把部分构造的对象暴露给新的线程。如果对象拥有一个线程，那么它应该提供一个启动该线程的 start() 或 init() 方法，而不是从构造器中启动它。

### 结束线程

&ensp;&ensp;&ensp;&ensp;线程会以以下三种方式之一结束：

 * 线程到达其 run() 方法的末尾。
 * 线程抛出一个未捕获到的 Exception 或 Error。
 * 另一个线程调用一个弃用的stop()方法。弃用是指这些方法仍然存在，但是您不应该在新代码中使用它们，并且应该尽量从现有代码中除去它们。 
当 Java 程序中的所有线程都完成时，程序就退出了。

### 加入线程

&ensp;&ensp;&ensp;&ensp;Thread API 包含了等待另一个线程完成的方法：join() 方法。当调用 Thread.join() 时，调用线程将阻塞，直到目标线程完成为止。

&ensp;&ensp;&ensp;&ensp;Thread.join() 通常由使用线程的程序使用，以将大问题划分成许多小问题，每个小问题分配一个线程。本章结尾处的示例创建了十个线程，启动它们，然后使用Thread.join() 等待它们全部完成。 

### 调度

&ensp;&ensp;&ensp;&ensp;除了何时使用 Thread.join() 和 Object.wait() 外，线程调度和执行的计时是不确定的。如果两个线程同时运行，而且都不等待，您必须假设在任何两个指令之间，其它线程都可以运行并修改程序变量。如果线程要访问其它线程可以看见的变量，如从静态字段（全局变量）直接或间接引用的数据，则必须使用同步以确保数据一致性。 

&ensp;&ensp;&ensp;&ensp;在以下的简单示例中，我们将创建并启动两个线程，每个线程都打印两行到 System.out

```java
public class TwoThreads { 
    public static class Thread1 extends Thread { 
    public void run() { 
        System.out.println("A"); 
        System.out.println("B"); 
    } 
 } 
    public static class Thread2 extends Thread { 
        public void run() { 
            System.out.println("1"); 
            System.out.println("2"); 
        } 
    } 
    public static void main(String[] args) { 
        new Thread1().start(); 
        new Thread2().start(); 
    } 
 } 
```
&ensp;&ensp;&ensp;&ensp;我们并不知道这些行按什么顺序执行，只知道“1”在“2”之前打印，以及“A”在“B”之前打印,输出可能是"12AB"、"1A2B"等等的任何一种。
     
### 休眠

&ensp;&ensp;&ensp;&ensp;Thread API 包含了一个sleep()方法，它将使当前线程进入等待状态，直到过了一段指定时间，或者直到另一个线程对当前线程的 Thread 对象调用了 Thread.interrupt()，从而中断了线程。当过了指定时间后，线程又将变成可运行的，并且回到调度程序的可运行线程队列中。 

&ensp;&ensp;&ensp;&ensp;如果线程是由对 Thread.interrupt() 的调用而中断的，那么休眠的线程会抛出InterruptedException，这样线程就知道它是由中断唤醒的，就不必查看计时器是否过期。 

&ensp;&ensp;&ensp;&ensp;Thread.yield() 方法就象 Thread.sleep() 一样，但它并不引起休眠，而只是暂停当前线程片刻，这样其它线程就可以运行了。在大多数实现中，当较高优先级的线程调用 Thread.yield() 时，较低优先级的线程就不会运行。 

&ensp;&ensp;&ensp;&ensp;CalculatePrimes 示例使用了一个后台线程计算素数，然后休眠十秒钟。当计时器过期后，它就会设置一个标志，表示已经过了十秒。

### 守护程序线程

&ensp;&ensp;&ensp;&ensp;我们提到过当 Java 程序的所有线程都完成时，该程序就退出，但这并不完全正确。隐藏的系统线程，如垃圾收集线程和由JVM创建的其它线程会怎么样？我们没有办法停止这些线程。如果那些线程正在运行，那么 Java 程序怎么退出呢？ 

&ensp;&ensp;&ensp;&ensp;这些系统线程称作守护程序线程。Java程序实际上是在它的所有非守护程序线程完成后退出的。 

&ensp;&ensp;&ensp;&ensp;任何线程都可以变成守护程序线程。可以通过调用 Thread.setDaemon() 方法来指明某个线程是守护程序线程。您也许想要使用守护程序线程作为在程序中创建的后台线程，如计时器线程或其它延迟的事件线程，只有当其它非守护程序线程正在运行时，这些线程才有用。 

### 例子

&ensp;&ensp;&ensp;&ensp;在这个示例中，TenThreads显示了一个创建了十个线程的程序，每个线程都执行一部分工作。该程序等待所有线程全部完成，然后收集结果。 

```java
public class TenThreads { 
    private static class WorkerThread extends Thread { 
    int max = Integer.MIN_VALUE; 
    int[] ourArray; 
    public WorkerThread(int[] ourArray) { 
        this.ourArray = ourArray; 
    } 
    // Find the maximum value in our particular piece of the array 
    public void run() { 
        for (int i = 0; i < ourArray.length; i++) 
            max = Math.max(max, ourArray[i]); 
        } 
        public int getMax() { 
            return max; 
        } 
    } 
    public static void main(String[] args) { 
        WorkerThread[] threads = new WorkerThread[10]; 
        int[][] bigMatrix = getBigHairyMatrix(); 
        int max = Integer.MIN_VALUE; 
 
        // Give each thread a slice of the matrix to work with 
        for (int i=0; i < 10; i++) { 
            threads[i] = new WorkerThread(bigMatrix[i]); 
             threads[i].start(); 
             } 
        // Wait for each thread to finish 
        try { 
             for (int i=0; i < 10; i++) { 
             threads[i].join(); 
             max = Math.max(max, threads[i].getMax()); 
            } 
        } 
        catch (InterruptedException e) { 
         // fall through 
        } 
        System.out.println("Maximum value was " + max); 
    } 
} 

```
### 小结

&ensp;&ensp;&ensp;&ensp;就象程序一样，线程有生命周期：它们启动、执行，然后完成。一个程序或进程也许包含多个线程，而这些线程看来互相单独地执行。 

&ensp;&ensp;&ensp;&ensp;线程是通过实例化 Thread 对象或实例化继承 Thread 的对象来创建的，但在对新的 Thread 对象调用 start() 方法之前，这个线程并没有开始执行。当线程运行到其 run() 方法的末尾或抛出未经处理的异常时，它们就结束了。 

&ensp;&ensp;&ensp;&ensp;sleep() 方法可以用于等待一段特定时间；而 join()方法可能用于等到另一个线程完成。

## 共享对数据的访问

### 受控访问的同步

&ensp;&ensp;&ensp;&ensp;为了确保可以在线程之间以受控方式共享数据，Java语言提供了两个关键字：synchronized 和volatile。 

&ensp;&ensp;&ensp;&ensp;Synchronized 有两个重要含义：它确保了一次只有一个线程可以执行代码的受保护部分（互斥，mutual exclusion或者说mutex），而且它确保了一个线程更改的数据对于其它线程是可见的（更改的可见性）。 

&ensp;&ensp;&ensp;&ensp;如果没有同步，数据很容易就处于不一致状态。例如，如果一个线程正在更新两个相关值（比如，粒子的位置和速率），而另一个线程正在读取这两个值，有可能在第一个线程只写了一个值，还没有写另一个值的时候，调度第二个线程运行，这样它就会看到一个旧值和一个新值。同步让我们可以定义必须原子地运行的代码块，这样对于其他线程而言，它们要么都执行，要么都不执行。 

&ensp;&ensp;&ensp;&ensp;同步的原子执行或互斥方面类似于其它操作环境中的临界段的概念。 

### 用锁保护的原子代码块

&ensp;&ensp;&ensp;&ensp;Volatile 对于确保每个线程看到最新的变量值非常有用，但有时我们需要保护比较大的代码片段，如涉及更新多个变量的片段。 

&ensp;&ensp;&ensp;&ensp;同步使用监控器（monitor）或锁的概念，以协调对特定代码块的访问。 

&ensp;&ensp;&ensp;&ensp;每个 Java 对象都有一个相关的锁。同一时间只能有一个线程持有 Java 锁。当线程进入synchronized代码块时，线程会阻塞并等待，直到锁可用，当它可用时，就会获得这个锁，然后执行代码块。当控制退出受保护的代码块时，即到达了代码块末尾或者抛出了没有在 synchronized 块中捕获的异常时，它就会释放该锁。 

&ensp;&ensp;&ensp;&ensp;这样，每次只有一个线程可以执行受给定监控器保护的代码块。从其它线程的角度看，该代码块可以看作是原子的，它要么全部执行，要么根本不执行。 

### 同步例子

&ensp;&ensp;&ensp;&ensp;使用 synchronized 块可以让您将一组相关更新作为一个集合来执行，而不必担心其它线程中断或看到计算的中间结果。以下示例代码将打印“1 0”或“01”。如果没有同步，它还会打印“1 1”（或“0 0”，随便您信不信）。 

```java
public class SyncExample { 
     private static lockObject = new Object(); 
     private static class Thread1 extends Thread { 
         public void run() { 
            synchronized (lockObject) { 
                x = y = 0; 
                System.out.println(x); 
            } 
        } 
     } 
    private static class Thread2 extends Thread { 
        public void run() { 
             synchronized (lockObject) { 
             x = y = 1; 
             System.out.println(y); 
            } 
        } 
    } 
    public static void main(String[] args) { 
     new Thread1().run(); 
     new Thread2().run(); 
    } 
} 
```

&ensp;&ensp;&ensp;&ensp;在这两个线程中都必须使用同步，以便使这个程序正确工作。

### Java 锁定

&ensp;&ensp;&ensp;&ensp;Java 锁定合并了一种互斥形式。每次只有一个线程可以持有锁。锁用于保护代码块或整个方法，必须记住是锁的身份保护了代码块，而不是代码块本身，这一点很重要。一个锁可以保护许多代码块或方法。 

&ensp;&ensp;&ensp;&ensp;反之，仅仅因为代码块由锁保护并不表示两个线程不能同时执行该代码块。它只表示如果两个线程正在等待相同的锁，则它们不能同时执行该代码。 

&ensp;&ensp;&ensp;&ensp;在以下示例中，两个线程可以同时不受限制地执行 setLastAccess() 中的 synchronized 块，因为每个线程有一个不同的 thingie 值。因此，synchronized 代码块受到两个正在执行的线程中不同锁的保护。 

```java
public class SyncExample { 
     public static class Thingie { 
         private Date lastAccess; 
         public synchronized void setLastAccess(Date date) { 
            this.lastAccess = date; 
         } 
     }
     public static class MyThread extends Thread { 
         private Thingie thingie; 
         public MyThread(Thingie thingie) { 
         this.thingie = thingie; 
         } 
        public void run() { 
            thingie.setLastAccess(new Date()); 
        } 
    } 
    public static void main() { 
         Thingie thingie1 = new Thingie(), 
         thingie2 = new Thingie(); 
         new MyThread(thingie1).start(); 
         new MyThread(thingie2).start(); 
     } 
} 
```

### 同步的方法

&ensp;&ensp;&ensp;&ensp;创建 synchronized 块的最简单方法是将方法声明成synchronized。这表示在进入方法主体之前，调用者必须获得锁： 

```java
public class Point { 
     public synchronized void setXY(int x, int y) { 
         this.x = x; 
         this.y = y; 
     } 
} 
```

&ensp;&ensp;&ensp;&ensp;对于普通的 synchronized 方法，这个锁是一个对象，将针对它调用方法。对于静态 synchronized 方法，这个锁是与 Class 对象相关的监控器，在该对象中声明了方法。 

&ensp;&ensp;&ensp;&ensp;仅仅因为 setXY() 被声明成 synchronized 并不表示两个不同的线程不能同时执行 setXY()，只要它们调用不同的 Point 实例的 setXY() 就可同时执行。对于一个 Point 实例，一次只能有一个线程执行 setXY()，或 Point 的任何其它 synchronized 方法。 
### 同步的块

&ensp;&ensp;&ensp;&ensp;synchronized 块的语法比synchronized方法稍微复杂一点，因为还需要显式地指定锁要保护哪个块。Point 的以下版本等价于前一页中显示的版本： 

```java


public class Point { 
     public void setXY(int x, int y) { 
         synchronized (this) { 
             this.x = x; 
             this.y = y; 
         } 
    } 
} 
```
&ensp;&ensp;&ensp;&ensp;使用 this 引用作为锁很常见，但这并不是必需的。这表示该代码块将与这个类中的 synchronized 方法使用同一个锁。 

&ensp;&ensp;&ensp;&ensp;由于同步防止了多个线程同时执行一个代码块，因此性能上就有问题，即使是在单处理器系统上。最好在尽可能最小的需要保护的代码块上使用同步。 

&ensp;&ensp;&ensp;&ensp;访问局部（基于堆栈的）变量从来不需要受到保护，因为它们只能被自己所属的线程访问。 

### 大多数类并没有同步

&ensp;&ensp;&ensp;&ensp;因为同步会带来小小的性能损失，大多数通用类，如 java.util 中的 Collection 类，不在内部使用同步。这表示在没有附加同步的情况下，不能在多个线程中使用诸如 HashMap 这样的类。 

&ensp;&ensp;&ensp;&ensp;通过每次访问共享集合中的方法时使用同步，可以在多线程应用程序中使用 Collection 类。对于任何给定的集合，每次必须用同一个锁进行同步。通常可以选择集合对象本身作为锁。

&ensp;&ensp;&ensp;&ensp;下一页中的示例类 SimpleCache显示了如何使用HashMap以线程安全的方式提供高速缓存。但是，通常适当的同步并不只是意味着同步每个方法。 

&ensp;&ensp;&ensp;&ensp;Collections 类提供了一组便利的用于 List、Map 和 Set 接口的封装器。您可以用Collections.synchronizedMap 封装Map，它将确保所有对该映射的访问都被正确同步。 
如果类的文档没有说明它是线程安全的，那么您必须假设它不是。 

### 示例：简单的线程安全的高速缓存

&ensp;&ensp;&ensp;&ensp;如以下代码样本所示，SimpleCache.java使用HashMap为对象装入器提供了一个简单的高速缓存。load() 方法知道怎样按对象的键装入对象。在一次装入对象之后，该对象就被存储到高速缓存中，这样以后的访问就会从高速缓存中检索它，而不是每次都全部地装入它。对共享高速缓存的每个访问都受到synchronized块保护。由于它被正确同步，所以多个线程可以同时调用 getObject 和clearCache 方法，而没有数据损坏的风险。 

```java
public class SimpleCache { 
     private final Map cache = new HashMap(); 
         public Object load(String objectName) { 
         // load the object somehow 
     } 
     public void clearCache() { 
         synchronized (cache) { 
            cache.clear(); 
         } 
     } 
     public Object getObject(String objectName) { 
         synchronized (cache) { 
             Object o = cache.get(objectName); 
             if (o == null) { 
                 o = load(objectName); 
                 cache.put(objectName, o); 
             } 
         } 
     return o; 
     } 
} 
```

### 小结

&ensp;&ensp;&ensp;&ensp;由于线程执行的计时是不确定的，我们需要小心，以控制线程对共享数据的访问。否则，多个并发线程会互相干扰对方的更改，从而损坏数据，或者其它线程也许不能及时看到对共享数据的更改。 

&ensp;&ensp;&ensp;&ensp;通过使用同步来保护对共享变量的访问，我们可以确保线程以可预料的方式与程序变量进行交互.

&ensp;&ensp;&ensp;&ensp;每个 Java 对象都可以充当锁，synchronized块可以确保一次只有一个线程执行由给定锁保护的synchronized 代码。 
