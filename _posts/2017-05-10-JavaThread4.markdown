---
layout:     post
title:      "Java并发编程 (五) - 线程通信"
subtitle:   "线程通信"
navcolor:   "invert"
date:       2017-05-10
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Java 
    - Thread 
---

# 线程通信

  线程运行时，运行顺序是由操作系统根据线程的优先级来决定的，同样优先级的线程，哪个先执行哪个后执行，我们是无法控制的，但是，有些情况下，需要按照一定的顺序或要求来执行线程，java提供了相应的一些机制，来协调这种情况，这就叫做线程通信。

  我们经常见到的需要线程通信的列子，如生产-消费者模型便是最经典的了，同时还有类似水池注水和出水，银行存款和取款等等一些都是需要线程通信。

## wait()，notify()和 notifyAll()

  这是传统的通信方式，这三个方法由Object类提供。

| 方法          | 作用                                       | 解释                                       |
| ----------- | ---------------------------------------- | ---------------------------------------- |
| wait()      | wait()方法使得当前线程必须要等待，等到另外一个线程调用notify()或者notifyAll()方法。 | 调用这个方法时，当前线程必须拥有对象监视器monitor。当该方法被调用后，monitor被释放，也就是对象锁被释放，线程处于Blocked状态，其他线程便可以使用monitor，直到其他线程利用notify()或notifyAll()发出通知，该线程便可以再次获得monitor从而获得执行机会，进入就绪状态。 |
| notify()    | notify()方法会唤醒**一个**等待当前对象的锁的线程。          | 如果有多个线程处于wait()一个monitor的Blocked状态，则该方法任意唤醒其中一个，让其进入就绪状态。 |
| notifyAll() | notifyAll()方法会唤醒**所有**等待当前对象的锁的线程。       | 所有被wait的线程，都进入就绪状态。                      |

> 这三个方法必须由同步监视器对象来调用。对于线程状态如Blocked、就绪，可参考[Java并发编程 (二) - 线程的生命周期和线程控制](http://www.betterming.cn/2017/04/19/JavaThread1/)

来看一个例子，银行提供存钱和取钱两种操作，我们让多人（多个线程）分别进行存取款操作，且顺序是存进去马上取，然后接着存，接着取，这就需要多个线程进行通信来完成。

Account类模拟银行账户：

```java
package com.example;

import java.util.concurrent.locks.Condition; //Condition类实现通信
import java.util.concurrent.locks.ReentrantLock;

public class Account {

    private String accountNO;
    private double balance;
    //标记位，false时无法取钱，true时无法存钱
    private boolean flag = false;
    //定义锁
    private final ReentrantLock lock = new ReentrantLock();
    //如果用lock显示锁，需用此类实现通信
    private final Condition condition = lock.newCondition();

    public Account() {

    }

    public Account(String accountNO, double balance) {
        this.accountNO = accountNO;
        this.balance = balance;
    }
  
    public String getAccountNO() {
        return accountNO;
    }

    public double getBalance() {
        return balance;
    }

    //线程安全的draw()方法，完成取钱操作
    //1、同步方法
    public synchronized void draw(double drawAmount){
    //2、显示锁的使用
//    public  void draw(double drawAmount) {
        //加锁
//        lock.lock();
        try {
            if (!flag) {
                wait();
//                condition.await();
            } else {
                System.out.println(Thread.currentThread().getName() +
                        "取钱成功，吐出钞票：" + drawAmount);
                balance -= drawAmount;
                System.out.println("\t余额为：" + balance);
                flag = false;
                notifyAll();
//                condition.signalAll();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            //释放锁
//            lock.unlock();
        }
    }

    //存钱方法
    public synchronized void deposit(double depositAmount) {
//        lock.lock();
        try {
            if (flag) {
                wait();
//                condition.await();
            } else {
                System.out.println(Thread.currentThread().getName() +
                        "存钱成功，存入：" + depositAmount);
                balance += depositAmount;
                System.out.println("\t余额为：" + balance);
                flag = true;
                notifyAll();
//                condition.signalAll();
            }

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
//            lock.unlock();
        }
    }

    //根据accountNO来重写hashCode()方法和equals()方法
    @Override
    public int hashCode() {
        return accountNO.hashCode();
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o != null && o.getClass() == Account.class) {
            Account target = (Account) o;
            return target.getAccountNO().equals(accountNO);
        }
        return false;
    }
}
```

模拟银行账户类，提供了存钱和取钱的方法，方法都为同步方法，保证账户钱数的线程安全性，定义一个标志位，来判断是否能够取钱或者存钱，以线程调用取钱方法draw()时为例，当flag为false，无法取钱，那么，调用wait();方法，释放同步监视器monitor，线程进入了Blocked状态，如果flag为true，便完成取钱操作，将flag至为true，然后调用notifyAll();让所有处于等待该monitor的线程进入就绪状态。

DrawThread类，模拟取钱线程。

```java
package com.example;

/**
 * Created by gaoming on 2017/4/11.
 */

public class DrawThread extends Thread {

    //模拟用户账户
    private Account account;
    //当前取钱线程要取的钱数
    private double drawAmount;

    public DrawThread(String name, Account account, double drawAmount){
        super(name);
        this.account = account;
        this.drawAmount = drawAmount;
    }

    //多个线程修改同一个共享数据，涉及数据安全问题
    @Override
    public void run() {
        for (int i = 0; i < 6; i++) {
            account.draw(drawAmount);
        }
    }
}
```

DepositTest类，模拟存钱线程。

```Java
package com.example;

/**
 * Created by gaoming on 2017/4/12.
 */

public class DepositTest extends Thread{

    //模拟用户账户
    private Account account;
    //当前线程要存的钱数
    private double depositAccount;

    public DepositTest(String name, Account account, double depositAccount){
        super(name);
        this.account = account;
        this.depositAccount = depositAccount;
    }

    @Override
    public void run() {
        for (int i = 0; i < 6; i++) {
            account.deposit(depositAccount);
        }
    }

}
```

DrawTest 测试主线程：

```java
package com.example;

/**
 * Created by gaoming on 2017/4/11.
 * 线程间通信，wait方法使线程进入阻塞状态，释放锁，通过notifiyall方法通知其他等待该锁的monitor访问该锁
 * 如果是显示lock同步加锁，则需用condition类来实现上面的功能
 */

public class DrawTest {

    public static void main(String[] args){
        Account acct = new Account("1234567",0);
        new DrawThread("取钱者 " , acct, 800).start();
        new DepositTest("存钱者甲 ", acct, 800).start();
        //new DepositTest("存钱者乙", acct, 800).start();
        //new DepositTest("存钱者丙", acct, 800).start();
    }
}
```

运行结果：

```
存钱者甲 存钱成功，存入：800.0
	余额为：800.0
取钱者 取钱成功，吐出钞票：800.0
	余额为：0.0
存钱者甲 存钱成功，存入：800.0
	余额为：800.0
取钱者 取钱成功，吐出钞票：800.0
	余额为：0.0
存钱者甲 存钱成功，存入：800.0
	余额为：800.0
取钱者 取钱成功，吐出钞票：800.0
	余额为：0.0
```

程序中，我们实例化了一个acct银行账户，然后，启动一个存钱线程和一个取钱线程，从运行结果来看，取钱线程和存钱线程分别交替进行，达到了我们线程通信的目的。

## Condition控制线程通信

  当我们使用ReentrantLock类显示加锁时，是通过ReentrantLock.lock()来加锁的，不同于synchronized获得了monitor，这样wait()，notify()和 notifyAll()方法便无法用了。于是，采用Condition类来实现通信。对应关系如下：

| synchronized 同步监视器对象 | Condition 显示加锁 |
| -------------------- | -------------- |
| wait()               | await()        |
| notify()             | signal()       |
| notifyAll()          | signalAll()    |

例子同样可以参考上面的Account类，注释掉的部分替换后，可以达到同样的效果。

## 阻塞队列BlockingQueue实现线程通信

  以上线程通信的方法有点类似于问答的形式，就像，两个或两个以上的线程通知其他一个或多个线程，说：我现在不需要执行了，你来执行，然后其他线程获得执行机会。BlockingQueue则类似于借助一个第三方数据结构来实现通信，多个都像这个队列来操作数据，从而达到按一定要求执行的顺序，完成线程的通信。

  已生产消费模型来看，定义如下：当生产者线程试图向**BlockingQueue**中放入元素时，如果该队列已满，则该线程被阻塞；当消费者试图从**BlockingQueue**中取出元素时，如果队列已空，则该线程被阻塞，因此来保证程序的两个线程通过交替向**BlockingQueue**中放入元素、取出元素，来控制线程同步，与此同时，生产者线程可以通过向队列中放入数据、消费者线程从队列取出数据，实现线程之间的数据传递。

BlockingQueue提供的方法如下：

|      | 阻塞线程   |
| ---- | ------ |
| 队尾插入 | put(e) |
| 队头删除 | take() |

ArrayBlockingQueue ：BlockingQueue是个继承自Queue的接口，ArrayBlockingQueue以数组的方式实现该队列。

生产消费者模型例子：

Producer类，生产者类，向队列放入元素。

```java
package com.example;

import java.util.concurrent.BlockingQueue;

/**
 * Created by gaoming on 2017/4/12.
 */

public class Producer implements Runnable{

    private BlockingQueue<String> bq;

    public Producer(){

    }
    public Producer(BlockingQueue<String> bq){
        this.bq = bq;
    }
    @Override
    public void run() {
        String[] strArr = new String[]{
                "java",
                "Struts",
                "Spring"
        };
        for (int i = 0;i < 3; i++){
//            System.out.println(Thread.currentThread().getName() +
//                    "生产者准备生产集合元素");
            try {
                Thread.sleep(2000);
                bq.put(strArr[i%3]);
                System.out.println(Thread.currentThread().getName() +
                        "队列放入：" + strArr[i%3]);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() +
                     "生产完成：");
        }
    }
}
```

Consumer 消费者类，向队列取出元素。

```java
package com.example;

import java.util.concurrent.BlockingQueue;

/**
 * Created by gaoming on 2017/4/12.
 */

public class Consumer implements Runnable {

    BlockingQueue<String> bq;

    public Consumer(BlockingQueue<String> bq) {
        this.bq= bq;
    }

    @Override
    public void run() {

        while (true){
//            System.out.println(Thread.currentThread().getName() +
//                    "消费者准备消费集合元素");
            try {
                Thread.sleep(200);
//                System.out.println(Thread.currentThread().getName() +
//                        "队列将要取出：" + bq);
                bq.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() +
                    "消费完成：");
        }
    }
}
```

测试类BlockingQueueTest，定义了一个ArrayBlockingQueue队列实例，以参数形式传入生产者和消费者，

```java
package com.example;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class BlockingQueueTest {
    public static void main (String [] args){
        BlockingQueue<String> bq = new ArrayBlockingQueue<String>(1);//队列大小为1，能存一个值
        Producer producer = new Producer(bq);
        Consumer consumer = new Consumer(bq);
        new Thread(producer,"生产者线程1 ").start();
//        new Thread(producer,"生产者线程2 ").start();
//        new Thread(producer,"生产者线程3 ").start();
        new Thread(consumer,"消费者线程 ").start();
    }
}
```

运行结果：

```
消费者线程 消费完成：
生产者线程1 队列放入：java
生产者线程1 生产完成：
生产者线程1 队列放入：Struts
生产者线程1 生产完成：
消费者线程 消费完成：
生产者线程1 队列放入：Spring
生产者线程1 生产完成：
消费者线程 消费完成：
```

从运行结果可以看到，生产和消费是交替进行的，这是因为，我们定义的队列大小为1，当消费者取出元素后，如果再取则是空的，所以消费者阻塞，生产者写入元素，当写入后，队列满了，生产者阻塞，于是这样交替进行，达到线程通信的目的。