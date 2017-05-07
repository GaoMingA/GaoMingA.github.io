---
layout:     post
title:      "Java并发编程 (三) - 线程安全 线程同步"
subtitle:   "线程安全 线程同步"
navcolor:   "invert"
date:       2017-04-23
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Java 
    - Thread 
---

# 线程同步

  要理解线程同步前，先要了解一个概念，那就是线程安全，什么是线程安全，什么是线程不安全呢？在说明阻塞的概念时，提及了排他锁，什么是排他锁？为什么线程要加排他锁？我们带着这些疑问一步一步分析。

## 线程安全

  线程安全指某个方法在多线程环境中被调用时，能够正确地处理多个线程之间的**共享变量**，使程序功能正确完成（摘自wikipedia）。以上是维基百科对多线程的概念描述，这句话的重点核心便是**共享变量**了，举个简单的例子来说明：

  实现一个银行账户的类Account ，Account这个类肯定有一个共享变量balance(账户金额)，假设这个balance上有1000元，如果有两个人，也就是两个线程同时通过网银去取800元钱，及时你在操作balance的方法 draw(balance）里面判断当钱数不够时，不能取出钱，这时依然有可能两个线程都取出钱，导致balance变成-200的情况，如下：

Account 账户类：

```java
/**
 * Created by gaoming on 2017/4/11.
 * 模拟银行账户
 */
public class Account {

    private String accountNO;
    private double balance;
    private boolean flag = false;
    //定义锁
    private final ReentrantLock lock = new ReentrantLock();

    public Account(){

    }
    public Account(String accountNO,double balance){
        this.accountNO = accountNO;
        this.balance = balance;
    }
  
    public String getAccountNO(){
        return accountNO;
    }

    public double getBalance(){
        return balance;
    }

    //线程安全的draw()方法，完成取钱操作
    //1、同步方法
    //public synchronized void draw(double drawAmount){
    //2、显示锁的使用
    public void draw(double drawAmount){
        //加锁
//        lock.lock();
        try {
            if (balance > drawAmount) {
                System.out.println(Thread.currentThread().getName() +
                        "取钱成功，吐出钞票：" + drawAmount);

                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                balance -= drawAmount;
                System.out.println("\t余额为：" + balance);

            } else {
                System.out.println(Thread.currentThread().getName() + "取钱失败，余额不足！");
            }
        }finally {
            //释放锁
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
        if (this == o){
            return true;
        }
        if (o != null && o.getClass() == Account.class){
            Account target = (Account)o;
            return target.getAccountNO().equals(accountNO);
        }
        return false;
    }
}
```

DrawThread类，模拟用户取钱

```java
/**
 * Created by gaoming on 2017/4/11.
 * 线程安全，同步方法，同步代码块，Lock显示锁的使用。
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
        //同步代码块结束，线程释放同步锁
        account.draw(drawAmount);
    }
}
```

Test类，启动两个线程同时取钱

```java
/**
 * Created by gaoming on 2017/4/11.
 */

public class DrawTest {

    public static void main(String[] args){
        Account acct = new Account("1234567",1000);
        new DrawThread("甲" , acct, 800).start();
        new DrawThread("乙" , acct, 800).start();
    }
}
```

运行结果：

```json
乙取钱成功，吐出钞票：800.0
甲取钱成功，吐出钞票：800.0
	余额为：200.0
	余额为：-600.0
```

  可以看到我们在draw()方法里面是加了判断的```if (balance > drawAmount) ```，但是，运行结果依然不是我们想要的结果，那么两个线程同时通过draw()方法操作了共享变量，得到了非预期结果，所以我们称作draw()这个方法是非线程安全的。那么反过来，如果能保证无论多少个线程调用draw()方法，我们都能得到想要的结果（balance不能小于0），那这个draw()方法便是线程安全的。

  为什么会出现上面线程不安全的情况呢，需要从[Java内存模型](www.betterming.cn)来分析。结下图合分析代码执行流程：

![java内存模型抽象图](https://github.com/GaoMingA/blogger/blob/master/img/java/JVMMemory.png?raw=true)

- 线程启动后，乙先获得执行机会，从main memory读取（balance 1000）到乙的local memory，这时乙的local memory 拷贝balance的本地副本（balance 1000），代码执行打印:```乙取钱成功，吐出钞票：800.0```，然后程序调用sleep()方法，线程甲便获得了执行机会，线程乙处于waitting状态。


- 线程甲从main memory中读取(balance 1000)到甲的local memory中，这时甲的local memory里面balance的本地副本（balance值也是1000），代码跳过判断 ```if (balance > drawAmount)```打印```甲取钱成功，吐出钞票：800.0```，接着程序调用sleep()方法，然后乙又获取了执行的机会，甲处于waitting状态。

- 乙获取到执行机会后，接着向下执行```balance -= drawAmount;```，打印```余额为：200.0```，这时，乙local memory中的balance值为200，这个值会更新到main memory中，乙这时执行完成，甲获得了执行机会。

- 甲会继续向下执行，但main memory中balance值变成了200 ，jvm会从main memory中拷贝副本到甲的local memory里面，所以甲的local memory中balance这时会更新到200，然后甲执行balance -= drawAmount;于是balance的值便得到了```余额为：-600.0```这条结果，然后从甲的local memory更新到main memory中去了。

  ​

  由于这种内存管理机制，所以造成了处理结果非想要的结果，如何解决这种问题呢？那就是，当乙从main memory获取到balance的拷贝值，直到乙执行完更新balance到main memory之前，甲都无法获取到balance并修改，那么这样就能保证同一时间只有一个线程修改共享变量，这就是线程同步。

## 线程同步

  上面的例子运行结果是我们不想要的，draw()方法是非线程安全的，那么，我们如何保证draw()方法是线程安全的呢，这里就要用到线程同步了，保证draw()方法在被调用的时候，其他线程无法使用draw()方法修改共享变量。

### synchronized

  java提供的关键字，用来给对象（obj）加一个排他锁，这个被加锁的对象也叫monitor（监视器），如下：

```Java
synchronized(obj){
  ...
}
```

  上面的关键字修饰后，obj只能被一个线程所拥有并执行，其他线程必须等这个线程释放该锁后才能获得执行机会，那么这样就能保证线程安全。接着上面的例子，看看draw()方法加了synchronized修饰后的执行。

```java
public synchronized void draw(double drawAmount){
        try {
            if (balance > drawAmount) {
                System.out.println(Thread.currentThread().getName() +
                        "取钱成功，吐出钞票：" + drawAmount);

                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                balance -= drawAmount;
                System.out.println("\t余额为：" + balance);

            } else {
                System.out.println(Thread.currentThread().getName() + "取钱失败，余额不足！");
            }
        }
}
```

运行结果如下：

```
甲取钱成功，吐出钞票：800.0
甲取钱后	余额为：200.0
乙取钱失败，余额不足！
```

可以看到加了synchronized 修饰该方法后执行结果是我们期望的结果，这是因为在线程调用```account.draw(drawAmount);```这个方法时，account对象的draw()方法被加锁了，甲线程无法在乙未更新balance之前去操作main memory里面的balance。这里synchronized修饰的虽然是方法，但是其监视的对象是account，其他对象调用draw()方法不会受影响，同样调用account的其他方法也是不被加锁的。

同步的加锁逻辑是：加锁 - 修改 - 释放锁。

### 同步锁类显示加锁

  java提供了一些类，这些类的对象可以用来显示的加锁，如：ReadWriteLock（读写锁）、ReentrantLock（可重入锁）等。上面的例子，我们用可重入锁来实现线程同步，如下：

```java
 //定义锁
    private final ReentrantLock lock = new ReentrantLock();
    ......
      
     public void draw(double drawAmount){
        //加锁
        lock.lock();
        try {
            if (balance > drawAmount) {
                System.out.println(Thread.currentThread().getName() +
                        "取钱成功，吐出钞票：" + drawAmount);

                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                balance -= drawAmount;
                System.out.println("\t余额为：" + balance);

            } else {
                System.out.println(Thread.currentThread().getName() + "取钱失败，余额不足！");
            }
        }finally {
            //释放锁
            lock.unlock();
        }
    }
```

运行结果：

```
甲取钱成功，吐出钞票：800.0
甲取钱后	余额为：200.0
乙取钱失败，余额不足！
```

在需要修改balance的地方，显示的加锁，结果依然达到了我们的预期结果，还是必须遵循加锁逻辑，结束后必须释放锁，否则会造成其他线程一直等待锁释放的情况，无法继续执行下去。

# 总结

  线程同步的目的，就是并发线程对共享资源的协调，以保证线程安全。