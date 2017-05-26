---
layout:     post
title:      "Java并发编程 (六) - 线程池1"
subtitle:   "线程池 ExecutorService ScheduledExecutorService"
navcolor:   "invert"
date:       2017-05-12
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Java 
    - Thread 
---

# 线程池

  [线程池](http://tutorials.jenkov.com/java-concurrency/thread-pools.html)（Thread Pool）顾名思义是一个池的概念，创建一个阻塞队列（阻塞队列概念可参考[Java并发编程 (五) - 线程通信](http://www.betterming.cn/2017/05/10/JavaThread4/)）来模拟池，当有一个并发任务（Runnable或Callable对象）需要线程执行时，我们将其加入线程池，池内有空闲线程时该并发任务就分配给该空闲线程处理。为什么要一个池来维护线程呢，因为考虑到同一时期多个[线程运行时的开销]()问题，利用线程池可以有效限制同一时刻运行的线程数量。

## ExecutorService 、ScheduledExecutorService

### 创建线程池

  Java5之后，developer不再需要自己写线程池了，Java内建支持线程池。ExecutorService是个继承自Executor的接口，ExecutorService实现的具体实例便是一个线程池，实现ExecutorService的类有：

- [ThreadPoolExecutor](http://tutorials.jenkov.com/java-util-concurrent/threadpoolexecutor.html)

- [ScheduledThreadPoolExecutor](http://tutorials.jenkov.com/java-util-concurrent/scheduledexecutorservice.html)

  ScheduledExecutorService是ExecutorService的子类，可以在指定延迟后执行线程任务。

  Executors类提供了很多种静态方法创建线程池，如下表为几种常用方法：

| 方法名                                      | 参数           | 概念   | 返回对象                     |
| ---------------------------------------- | ------------ | ---- | ------------------------ |
| newCachedThreadPool()                    | 无            |      | ExecutorService          |
| newFixedThreadPool(int nThreads)         | nThreads     |      | ExecutorService          |
| newScheduledThreadPool(int corePoolSize) | corePoolSize |      | ScheduledExecutorService |
| newSingleThreadExecutor()                | 无            |      | ExecutorService          |

下面是几个创建例子：

```java
ExecutorService executorService1 = Executors.newSingleThreadExecutor();

ExecutorService executorService2 = Executors.newFixedThreadPool(10);

ExecutorService executorService3 = Executors.newScheduledThreadPool(10);
```

本质上上面的方法返回了`ExecutorService`对象，是通过`ThreadPoolExecutor`类new出来的实例，也就是ExecutorService接口的具体实现类的实例，下面是`newFixedThreadPool`源码，可以看到，内部是通过链试阻塞队列实现的：

```java
public static ExecutorService newFixedThreadPool(int var0) {
        return new ThreadPoolExecutor(var0, var0, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue());
    }
```

```java
 ThreadPoolExecutor(int corePoolSize, 
                    int maximumPoolSize,
                    long keepAliveTime,
                    TimeUnit.MILLISECONDS, 
                    new LinkedBlockingQueue());
```
1. corePoolSize 线程池基本大小必须大于或等于0；
2. maximumPoolSize 线程池最大容量必须大于或等于1；
3. keepAliveTime 线程存活保持时间必须大于或等于0；
4. LinkedBlockingQueue 任务队列；

在用newFixedThreadPool(int var0) 这里传入参数是被corePoolSize和maximumPoolSize用的，所以其创建的线程池基本容量和最大容量都为var0，既这个线程池是固定容量的，超出容量的线程会在LinkedBlockingQueue队列中等待。

**总之，`ExecutorService`是个接口，`ThreadPoolExecutor`是其具体实现类，`Executors`提供的静态方法是根据不同需求实现的`ThreadPoolExecutor`对象，从而可以用`Executors`的不同方法创建不同要求的线程池。**

创建线程池后，可以用```execute(Runnable)```、```submit(Runnable)```、`submit(Callable)`等方法将Runnable或Callable对象提交给线程池，由线程池维护的线程执行并发任务。下面是个通过```newFixedThreadPool()```创建线程池的例子。

```java
package com.example;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolTest {

    public static void main(String[] args){
        //创建一个线程池pool 具有固定线程数3
        ExecutorService pool = Executors.newFixedThreadPool(3);
//        ExecutorService pool = Executors.newCachedThreadPool();
        Runnable task = new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 3 ;i++) {
                    System.out.println(Thread.currentThread().getName() +
                            " i的值为:" + i);
                }
            }
        };
        pool.submit(task);
        pool.submit(task);
        //pool.submit(task);
        pool.shutdown();

    }
}
```

执行结果：

```java
pool-1-thread-1 i的值为:0
pool-1-thread-2 i的值为:0
pool-1-thread-1 i的值为:1
pool-1-thread-1 i的值为:2
pool-1-thread-2 i的值为:1
pool-1-thread-2 i的值为:2
```
&emsp;&emsp;上面的例子创建了一个固定线程数为3的线程池，将两个并发打印任务通过submit提交给线程池，从执行结果来看，pool-1中分别维护了thread-2和thread-1两个线程，并执行了其打印任务，线程池最后通过`shutdown()`进行关闭。

### 向线程池提交并发任务

上面的例子通过`ExecutorService pool = Executors.newFixedThreadPool(3);`创建了一个固定线程个数为3的线程池pool，然后将Runnable对象task通过`submit(Runnable)`方法递交给了线程池，由线程池来管理并行任务。那么向线程池递交并发任务的方式有哪些呢，如下表：

| 方法                                       | 返回值                                      | 作用线程池                    |
| ---------------------------------------- | ---------------------------------------- | ------------------------ |
| execute(Runnable)                        | 无                                        | ExecutorService          |
| submit(Runnable)                         | 返回Future对象，这个Future对象可以用于检测Runnable run()是否执行结束   `future.get()`;  //returns null if the task has finished correctly. | ExecutorService          |
| submit(Callable)                         | 返回Future对象， `future.get()` ；Callable中call()方法的返回值 | ExecutorService          |
| schedule (Callable task, long delay, TimeUnit timeunit) | 指定Callable任务将在delay延迟后执行，ScheduledFuture 对象`scheduledFuture.get()`获得Callable返回值 | ScheduledExecutorService |
| schedule (Runnable task, long delay, TimeUnit timeunit) | 无返回值， `ScheduledFuture.get()` method 返回 null 当任务结束的时候. | ScheduledExecutorService |
| scheduleAtFixedRate (Runnable, long initialDelay, long period, TimeUnit timeunit) | initialDelay 时间每个period开始执行，无返回值         | ScheduledExecutorService |
| scheduleWithFixedDelay (Runnable, long initialDelay, long period, TimeUnit timeunit) | In this method, however, the `period` is interpreted as the delay between the **end** of the previous execution, until the start of the next. | ScheduledExecutorService |

### 关闭线程池

当不再使用线程池的时候应该关闭线程池，如果不关闭的话，即使没有线程执行，线程池依然会在JVM中保持Running。关闭线程池的方法如下：

- `shutdown()`  不会立即关闭线程池，线程池不再接受新的线程任务，会等线程池中的线程都执行结束后关闭线程池。

- `shutdownNow()`  将立即停止所有执行任务，并跳过所有已提交但未处理的任务。对执行任务没有任何保证，也许他们停止，也许执行直到结束。

只要调用了这两个关闭方法的其中一个，isShutdown方法就会返回true。当所有的任务都已关闭后,才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于我们应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow。

## ForkJoinPool



以下内容转载自：[聊聊并发（三）——JAVA线程池的分析和使用](http://www.infoq.com/cn/articles/java-threadPool)

## 线程池线程执行流程

![线程池处理流程](https://github.com/GaoMingA/blogger/blob/master/img/java/threadpool.jpg?raw=true)

从上图我们可以看出，当提交一个新任务到线程池时，线程池的处理流程如下：

1. 首先线程池判断**基本线程池**是否已满？没满，创建一个工作线程来执行任务。满了，则进入下个流程。

2. 其次线程池判断**工作队列**是否已满？没满，则将新提交的任务存储在工作队列里。满了，则进入下个流程。

3. 最后线程池判断**整个线程池**是否已满？没满，则创建一个新的工作线程来执行任务，满了，则交给饱和策略来处理这个任务。

**源码分析**。上面的流程分析让我们很直观的了解了线程池的工作原理，让我们再通过源代码来看看是如何实现的。线程池执行任务的方法如下：

```java
public void execute(Runnable command) {
    if (command == null)
       throw new NullPointerException();
    //如果线程数小于基本线程数，则创建线程并执行当前任务 
    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
    //如线程数大于等于基本线程数或线程创建失败，则将当前任务放到工作队列中。
        if (runState == RUNNING && workQueue.offer(command)) {
            if (runState != RUNNING || poolSize == 0)
                      ensureQueuedTaskHandled(command);
        }
    //如果线程池不处于运行中或任务无法放入队列，并且当前线程数量小于最大允许的线程数量，
则创建一个线程执行任务。
        else if (!addIfUnderMaximumPoolSize(command))
        //抛出RejectedExecutionException异常
            reject(command); // is shutdown or saturated
    }
}
```

**工作线程**。线程池创建线程时，会将线程封装成工作线程Worker，Worker在执行完任务后，还会无限循环获取工作队列里的任务来执行。我们可以从Worker的run方法里看到这点：

```java
public void run() {
     try {
           Runnable task = firstTask;
           firstTask = null;
            while (task != null || (task = getTask()) != null) {
                    runTask(task);
                    task = null;
            }
      } finally {
             workerDone(this);
      }
} 
```

## 线程池的使用原则

要想合理的配置线程池，就必须首先分析任务特性，可以从以下几个角度来进行分析：

1. 任务的性质：CPU密集型任务，IO密集型任务和混合型任务。
2. 任务的优先级：高，中和低。
3. 任务的执行时间：长，中和短。
4. 任务的依赖性：是否依赖其他系统资源，如数据库连接。

任务性质不同的任务可以用不同规模的线程池分开处理。CPU密集型任务配置尽可能小的线程，如配置Ncpu+1个线程的线程池。IO密集型任务则由于线程并不是一直在执行任务，则配置尽可能多的线程，如2*Ncpu。混合型的任务，如果可以拆分，则将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐率要高于串行执行的吞吐率，如果这两个任务执行时间相差太大，则没必要进行分解。我们可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。

优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先得到执行，需要注意的是如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。

执行时间不同的任务可以交给不同规模的线程池来处理，或者也可以使用优先级队列，让执行时间短的任务先执行。

依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，如果等待的时间越长CPU空闲时间就越长，那么线程数应该设置越大，这样才能更好的利用CPU。

建议使用有界队列，有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点，比如几千。有一次我们组使用的后台任务线程池的队列和线程池全满了，不断的抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞住，任务积压在线程池里。如果当时我们设置成无界队列，线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然我们的系统所有的任务是用的单独的服务器部署的，而我们使用不同规模的线程池跑不同类型的任务，但是出现这样问题时也会影响到其他任务。

## 线程池的监控

通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用

- taskCount：线程池需要执行的任务数量。
- completedTaskCount：线程池在运行过程中已完成的任务数量。小于或等于taskCount。
- largestPoolSize：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过。如等于线程池的最大大小，则表示线程池曾经满了。
- getPoolSize:线程池的线程数量。如果线程池不销毁的话，池里的线程不会自动销毁，所以这个大小只增不+ getActiveCount：获取活动的线程数。

通过扩展线程池进行监控。通过继承线程池并重写线程池的beforeExecute，afterExecute和terminated方法，我们可以在任务执行前，执行后和线程池关闭前干一些事情。如监控任务的平均执行时间，最大执行时间和最小执行时间等。这几个方法在线程池里是空方法。如：

```java
protected void beforeExecute(Thread t, Runnable r) { }
```