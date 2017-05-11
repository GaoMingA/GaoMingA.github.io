---
layout:     post
title:      "Java并发编程 (四) - 线程死锁"
subtitle:   "线程死锁"
navcolor:   "invert"
date:       2017-05-05
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Java 
    - Thread 
---

# 线程死锁

## 概念

  两个或者两个以上的线程都在等待对方释放同步监视器时，没有一个线程释放同步监视器，导致所有线程一直处于Blocked状态，这样的情况就产生了死锁。

## 示例

```java
package com.example;

class A {

    public synchronized void foo(B b){

        System.out.println("当前线程名：" + Thread.currentThread().getName()
        + "进入了A实例的foo方法");

        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("当前线程名：" + Thread.currentThread().getName()
                + "企图进入B实例的last方法");
        b.last();
    }

    public synchronized void last() {
        System.out.println("进入了A类的last方法内部");
    }
}
```

```java
package com.example;

/**
 * Created by gaoming on 2017/4/11.
 */

class B {
    public synchronized void bar(A a){

        System.out.println("当前线程名：" + Thread.currentThread().getName()
                + "进入了B实例的bar方法");

        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("当前线程名：" + Thread.currentThread().getName()
                + "企图进入A实例的last方法");
        a.last();
    }

    public synchronized void last() {
        System.out.println("进入了B类的last方法内部");
    }
}

```

```java
package com.example;

/**
 * Created by gaoming on 2017/4/11.
 * 启动两个线程分别锁定A和B的实例，然后A的实例又请求锁定B，B又请求锁定A，于是产生死锁。
 */

public class DeadLock implements Runnable {

    A a = new A();
    B b = new B();

    public void init(){
        Thread.currentThread().setName("主线程 ");
        a.foo(b);
        System.out.println("进入了主线程后");
    }

    @Override
    public void run() {
        Thread.currentThread().setName("副线程 ");
        b.bar(a);
        System.out.println("进入了副线程后");
    }

    public static void main(String[] args){
        DeadLock deadLock = new DeadLock();
        new Thread(deadLock).start();
        deadLock.init();
    }
}
```

```
当前线程名：主线程 进入了A实例的foo方法
当前线程名：副线程 进入了B实例的bar方法
当前线程名：副线程 企图进入A实例的last方法
当前线程名：主线程 企图进入B实例的last方法
```

如上例子，```new Thread(deadLock).start();```启动了线程DeadLock，然后主线程对象deadLock调用init()方法，由打印结果可以看到， ```当前线程名：主线程 进入了A实例的foo方法```,虽然先启动了线程，然后再运行的init()方法，但系统给init()方法优先的执行机会，这时，主线程```a.foo(b);```,被调用，打印上面的log，调用这个方法时，a对象被加锁，主线程持有锁a。接着，```Thread.sleep(200);```，deadLock启动的线程获得执行机会，```b.bar(a);```这句被调用，打印```当前线程名：副线程 进入了B实例的bar方法```，对象b被加锁，副线程持有锁b。这里，a和b两个对象monitor都被加锁了。接着副线程继续执行```a.last();```，在这之前打印了```当前线程名：副线程 企图进入A实例的last方法```，这里，a对象企图调用在A类中实现的last()方法，这个last()方法是个同步方法，于是要申请对象锁a，发现a已经是被加锁的（主线程持有），于是，副线程阻塞Blocked，主线程获得执行机会，接着上面的代码执行到```b.last();```，b对象要调用B类中的last()这个同步方法，需要申请锁b，然而b对象被副线程持有，副线程Blocked。于是程序便进入了死锁状态。

这么绕，看下图：

![示例死锁流程](https://github.com/GaoMingA/blogger/blob/master/img/java/DeadLock.png?raw=true)

# 总结

死锁产生后程序一直处于Blocked情况，这种情况是developer不想看到的，代码中要劲量避免死锁，但也不免碰到死锁问题，需要我们去处理。Android系统中，如果出现死锁，当主线程等待超过5s时，便出现ANR，我们可以通过相关trace文件分析打印的堆栈信息进一步确定死锁是在哪些进程或线程产生的。

