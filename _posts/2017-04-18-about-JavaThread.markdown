---
layout:     post
title:      "Java并发编程 (一)"
subtitle:   "总纲 线程的创建和启动"
navcolor:   "invert"
date:       2017-04-18
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Java 
    - Thread 
---

# 概述

  首先，如下图，列出个大纲，我们将所有的知识点，通过思维导图引导出来，然后根据每个知识点详细展开，以后碰到并发中的新问题或知识点，都可添加到大纲，不断完善文档。

![线程知识点大纲](https://github.com/GaoMingA/blogger/blob/master/img/java/ThreadOutline.png?raw=true)

# 一、线程的创建和启动

  线程创建的方式主要有三种，分别是继承自Thread类，实现Runnable接口，实现Callable接口。

## 2.1 继承Thread类

  看如下列子，定义一个ClassThread类，继承自Thread类，重写Thread类的run()方法，实现既该线程需要做的内容。主函数main()直接通过ClassThread的对象classThread1和classThread2调用start()方法启动了两个线程。

```Java
/**
 * author : gaoming
 * 目的: 创建线程类，继承自Thread，变量 i 不能共享，该类对象可以直接启动线程
 */

public class ClassThread extends Thread{
    private int i;
    public void run (){
        for (i = 0; i < 3 ; ++i){
            System.out.println(Thread.currentThread().getName()+" "+i);
        }
    }

    public static void main(String[] args) {
        ClassThread classThread1 = new ClassThread();
        ClassThread classThread2 = new ClassThread();
        for (int i = 0; i < 3 ; i++){
            System.out.println(Thread.currentThread().getName()+" "+i);
            if (i == 2){
//                new ClassThread().start();
//                new ClassThread().start();
                classThread1.start();
                classThread2.start();
                //System.out.println(classThread1.isAlive());
            }
        }
    }
}
```

  上面的例子很简单，在run()方法里面只打印一个全局变量i，我们看如下一次的运行结果：

```
main 0
main 1
main 2
Thread-0 0
Thread-1 0
Thread-1 1
Thread-0 1
Thread-0 2
Thread-1 2
```

  分析一下运行结果，从main线程到2时，开始启动两个线程顺序打印变量全局变量i，可以看到Thread-0 和 Thread-1在同时打印i这个变量，且打印的值是各自打印各自的。也就是说，继承自Thread类的线程类，不会共享这个公共资源i。这与继承接口实现的类是不一样的。

## 2.2 实现Runnable接口

  同2.1一样，现在将2.1的例子以Runnable接口来实现，ClassThreadRunnable类，实现Runnable接口，实现接口方法run()，main()方法中new一个该类的实例classThreadRunnable，new两个线程Thread，将该实例以参数的形式传入，来启动线程。

```java
/**
 * author : gaoming
 * 目的: 创建线程类，实现Runnable接口，共享变量 i，接口启动线程必须通过Thread类，传入参数
 */

public class ClassThreadRunnable implements Runnable{
    private int i=0;
    @Override
    public void run() {
        for (; i < 3 ; i++){
            System.out.println(Thread.currentThread().getName()+" "+i);
        }
    }

    public static void main(String[] args){
        for (int i = 0; i < 3 ; i++){
            System.out.println(Thread.currentThread().getName()+" "+i);
            if (i == 2){
                ClassThreadRunnable classThreadRunnable = new ClassThreadRunnable();
                new Thread(classThreadRunnable).start();
                new Thread(classThreadRunnable).start();
            }
        }
    }
}
```

  执行结果如下：

```
main 0
main 1
main 2
Thread-0 0
Thread-0 1
Thread-0 2
```

  可以和2.1的运行结果对比，Thread-1 为什么没有打印值，因为我们的循环比较少只有三次，cpu还没来的及给Thread-1 执行的机会，任务就完成了（不一定每次是这样）。这根2.1对比的关键就是，2.1中，无论如何都会执行Thread-1 线程，因为在2.1中 资源i不是共享的，两个线程各自做各自的打印任务，而2.2中资源i是共享资源了，两个线程有一个共同目标就是完成i的打印，所以Thread-0 做完这个目标后，Thread-1 就不会参与了。这就是两种实现线程的方式的最主要区别—继承自Thread类的线程类，多个线程不共享资源，实现接口Runnable的线程类共享资源。共享资源的线程是不安全的，后面我们在线程安全一节详细讨论。

> 注：启动线程必须使用start()方法，而不能直接使用run()方法，如果是使用了run()方法，那么会吧run()方法当做一个普通方法去执行，而不是一个并发的线程。

## 2.3 Callable、Future和FutureTask

  实现Callable接口的线程类可以有返回值，配合Future实现。Future是个接口，提供了方法来控制Callable的执行结果，包括取消执行，判断任务是否完成，获取执行结果等：

```java
boolean cancel(boolean var1); //取消执行

boolean isCancelled(); //判断是否被取消

boolean isDone(); //判断是否被执行完成

V get() throws InterruptedException, ExecutionException; //获取执行结果

V get(long var1, TimeUnit var3) throws InterruptedException, ExecutionException, TimeoutException; //获取执行结果，指定时间内未获取返回null
```

  FutureTask是Future接口的一个具体实现，同时还实现了Runnable接口。

#### 2.3.1 Callable FutureTask

  我们首先看一个Callable配合FutureTask的例子，CallableTask类实现了Callable<>接口，Callable接口实现方法call()，这点不同于Runnable和继承Thread类的实现，在call()方法中，计算了0-99的整数和，返回结果sum。CallableThreadTest是我们例子写的类用来启动这个线程并获取线程的结果，配合FutureTask使用，代码如下：

```java
/**
 * Created by gaoming on 2017/4/19.
 * 目的：Callable接口实现的线程类返回结果，通过FutureTask获取返回的结果
 */

public class CallableThreadTest {

    public static void main(String[] args){
        //Callable实现的线程类的启动
        CallableTask callableTask = new CallableTask();
        FutureTask<Integer> futureTask = new FutureTask<>(callableTask);
        new Thread(futureTask).start();
        try {
            int result = futureTask.get(); //获取线程执行结果
            System.out.println(Thread.currentThread().getName() +
                    "收到的结果是：" + result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class CallableTask implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        int sum = 0;
        for (int i = 0; i < 100; i++){
            sum += i;
        }
        System.out.println(Thread.currentThread().getName() +
                "计算的结果是：" + sum);
        return sum;
    }
}
```

  执行结果如下：

```
Thread-0计算的结果是：4950
main收到的结果是：4950
```

  可以看到，我们打印的收到的结果和线程中计算的结果一样，说明我们在主线程里面收到了Thread-0线程的返回值。

#### 2.3.2 Callable Future

  还是上面的列子，我们使用Future接口来直接获取Callable线程类的执行结果：

```Java
/**
 * Created by gaoming on 2017/4/19.
 * 目的：Callable接口实现的线程类返回结果，通过Future获取返回的结果
 */
public class CallableThreadTest1 {
    public static void main(String[] args){
        //Callable实现的线程类的启动
        ExecutorService executorService = Executors.newFixedThreadPool(1);
        CallableTask callableTask = new CallableTask();
        Future<Integer> future = executorService.submit(callableTask);
        executorService.shutdown();
        /**
        // FutureTask 用线程池管理，获取线程执行结果
        ExecutorService executorService = Executors.newFixedThreadPool(1);
        CallableTask callableTask = new CallableTask();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(callableTask);
        executorService.submit(futureTask);
        executorService.shutdown();
        **/
        try {
            //int result = futureTask.get();
            int result = future.get(); //获取线程执行结果
            System.out.println(Thread.currentThread().getName() +
                    "收到的结果是：" + result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

  上面的类CallableThreadTest1 执行结果和 2.3.1中的类CallableThreadTest是一样的，只是获取线程返回值是通过Future，这里用到了ExecutorService这个类，是实现了一个线程池，将callableTask交给线程池管理，执行结果交由future对象管理，因为Future是个接口，不是一个实例化的类，所以不能像futureTask那样直接用future启动线程。

# 总结

  要实现多线程并发编程，有三种实现方式，分别是继承Thread类，实现Runnable和Callable接口。三种方式对比如下：

|              | 任务方法    | 安全性   | 易用性  | 扩展性     | 返回值  |
| ------------ | ------- | :---- | ---- | ------- | ---- |
| 继承Thread类    | run();  | 资源不共享 | 简单   | 不能继承其他类 | 无    |
| 实现Runnable接口 | run();  | 资源共享  | 较复杂  | 可以继承其他类 | 无    |
| 实现Callable接口 | call(); | 资源共享  | 复杂   | 可以继承其他类 | 有    |

  在实际使用中，可以考虑上述因素来选择线程的实现方式。