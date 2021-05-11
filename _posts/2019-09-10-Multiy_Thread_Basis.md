---
layout: post
title: 'Java 多线程基础'
date: 2019-09-10
author: yemong.gao
cover: '/images/2019-09-10-Multiy_Thread_Basis/image00.jpg'
tags: java 多线程
---

> 实现线程的方式、线程状态、JMM模型、Runnable与Callable的区别、线程池

# Java 多线程

### 实现线程的方式

* **1，继承Thread**
```java
public class MyThread extends Thread {

    @Override
    public void run() {
        //TODO
        System.out.println("继承Thread : " + Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        Thread t = new MyThread();
        t.start();
    }
}
```
* **2，实现Runnable**
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        //TODO
        System.out.println("实现Runnable : "+Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable());
        t.start();
    }
}
```
* **3，实现Callable<V>  jdk1.5之后**
```java
public class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        //TODO
        System.out.println("实现Callable : "+Thread.currentThread().getName());
        return 1;
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        //创建MyThread实例
        Callable<Integer> c = new MyCallable();
        //获取FutureTask
        FutureTask<Integer> ft = new FutureTask<>(c);
        //使用FutureTask初始化Thread
        Thread t = new Thread(ft);
        t.start();
        System.out.println("获取call()返回值 : "+ ft.get());
    }
}
```
* **4，使用线程池创建**
```java
public class MyPool {

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        executorService.execute(new MyThread());
        executorService.execute(new MyRunnable());
        FutureTask<Integer> ft = new FutureTask<>(new MyCallable());
        executorService.submit(ft);
        executorService.shutdown();
    }
}
```

### Runnable与Callable的区别

* Runnable与Callable都需要调用Thread.start()启动线程；
* Runnable是在JDK1.0的时候提出来的多线程的实现接口，而Callable是在JDK1.5之后提出的多线程的实现接口
* java.lang.Runable接口之中只提供有一个run()方法，并且没有返回值；
* java.util.concurrent.Callable接口提供call()方法，可以有返回值；可以通过FutureTask中的get()方法获取返回值(`FutureTask.get()方法实现，此方法会阻塞主线程直到获取‘将来’结果；当不调用此方法时，主线程不会阻塞！`)

* java.lang.Runable接口的run()方法的异常只能在方法内部处理，不能上抛。
* java.util.concurrent.Callable接口提供call()方法允许抛出异常

![image01](/images/2019-09-10-Multiy_Thread_Basis/image01.png)

### 线程池ExcutorService中的excutor和submit方法的区别

* 1、excutor没有返回值，submit有返回值，并且返回执行结果Future对象
* 2、excutor不能提交Callable任务，只能提交Runnable、Thread任务，submit两者任务都可以提交
* 3、在submit中提交Runnable、Thread任务，会返回执行结果Future对象，但是Future调用get方法将返回null（Runnable没有返回值）

### ExecutorCompletionService

当我们用线程池submit提交一组多线程任务，当我们需要每个任务完成时并获取它的返回值；有两种方式可以采取；第一种使用List获取保存返回值，第二种使用ExecutorCompletionService获取保存返回值。

**示例代码List与ExecutorCompletionService比对**
```java
package com.yeming.spring.example.thread;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

/**
 * @author yeming.gao
 * @Description: ExecutorCompletionService获取一组任务示例
 * @date 2020/10/21 14:23
 */
public class ExecutorCompletionServiceTest {

    static class CallableTask implements Callable<String> {
        private int i;

        public CallableTask(int i) {
            this.i = i;
        }

        @Override
        public String call() throws Exception {
            Thread.sleep(20000);
            return Thread.currentThread().getName() + "执行完任务：" + i;
        }
    }


    public static void main(String[] args) {
        int taskSize = 5;

        System.out.println("ExecutorService start...");
        long begin01 = System.currentTimeMillis();
        // 创建一个线程池
        ExecutorService pool01 = Executors.newFixedThreadPool(taskSize);
        ExecutorCompletionService<String> completionService = new ExecutorCompletionService<>(pool01);
        try {
            for (int i = 1; i <= taskSize; i++) {
                CallableTask callableTask = new CallableTask(i);
                completionService.submit(callableTask);
            }

            for (int i = 1; i <= taskSize; i++) {
                System.out.println(completionService.take().get());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        // 关闭线程池
        pool01.shutdown();

        long end01 = System.currentTimeMillis();
        System.out.println("ExecutorService end..excute time：" + (end01 - begin01) + "ms");

        System.out.println("-------------------------------------------------------");

        System.out.println("List start...");
        long begin02 = System.currentTimeMillis();
        // 创建一个线程池
        ExecutorService pool02 = Executors.newFixedThreadPool(taskSize);
        // 创建多个返回值的任务
        List<Future<String>> list = new ArrayList<>();
        try {
            for (int i = 1; i <= taskSize; i++) {
                CallableTask callableTask = new CallableTask(i);
                Future<String> future = pool02.submit(callableTask);
                list.add(future);
            }
            for (Future<String> aList : list) {
                System.out.println(aList.get());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        // 关闭线程池
        pool02.shutdown();

        long end02 = System.currentTimeMillis();
        System.out.println("List end..excute time：" + (end02 - begin02) + "ms");
    }
}

/**
* 程序运行结果：
*
ExecutorService start...
pool-1-thread-3执行完任务：3
pool-1-thread-4执行完任务：4
pool-1-thread-5执行完任务：5
pool-1-thread-2执行完任务：2
pool-1-thread-1执行完任务：1
ExecutorService end..excute time：20011ms
-------------------------------------------------------
List start...
pool-2-thread-1执行完任务：1
pool-2-thread-2执行完任务：2
pool-2-thread-3执行完任务：3
pool-2-thread-4执行完任务：4
pool-2-thread-5执行完任务：5
List end..excute time：20002ms
**/
```
这里使用了线程池提高了整体的执行效率，但遍历这些Future，调用Future接口实现类的get方法是阻塞的，也就是和当前这个Future关联的计算任务真正执行完成的时候，get方法才返回结果，如果当前计算任务没有执行完成，而有其它Future关联的计算任务已经执行完成了，就会白白浪费很多等待的时间，所以最好是遍历的时候谁先执行完成就先获取哪个结果，这样就节省了很多持续等待的时间。

而ExecutorCompletionService可以实现这样的效果，它的内部有一个先进先出的阻塞队列，用于保存已经执行完成的Future，通过调用它的take方法或poll方法可以获取到一个已经执行完成的Future，进而通过调用Future接口实现类的get方法获取最终的结果。

**ExecutorCompletionService方法解析**
```java
public class ExecutorCompletionService<V> implements CompletionService<V> {
	...
}
```
```java
public interface CompletionService<V> {

    Future<V> submit(Callable<V> task);

    Future<V> submit(Runnable task, V result);

    Future<V> take() throws InterruptedException;

    Future<V> poll();

    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}
```
`ExecutorCompletionService实现了CompletionService接口，在CompletionService接口中定义了如下这些方法：`

* Future<V> submit(Callable<V> task):提交一个Callable类型任务，并返回该任务执行结果关联的Future；
* Future<V> submit(Runnable task,V result):提交一个Runnable类型任务，并返回该任务执行结果关联的Future；
* Future<V> take():从内部阻塞队列中获取并移除第一个执行完成的任务，阻塞，直到有任务完成；
* Future<V> poll():从内部阻塞队列中获取并移除第一个执行完成的任务，获取不到则返回null，不阻塞；
* Future<V> poll(long timeout, TimeUnit unit):从内部阻塞队列中获取并移除第一个执行完成的任务，阻塞时间为timeout，获取不到则返回null；

### 线程状态

线程一共有六种状态，分别为New、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED；具体状态信息可以看Thread类中的State枚举类。
`同一时刻只有一种状态，通过线程的getState方法可以获取线程的状态。`

```java
public enum State {
    /**
     * 当线程被创建出来还没有被调用start()时候的状态
     */
    NEW,

    /**
     * 当线程被调用了start()，且处于等待操作系统分配资源（如CPU）、等待IO连接、正在运行状态，
	 * 即表示Running状态和Ready状态。
     * 
     * 注：不一定被调用了start()立刻会改变状态，还有一些准备工作，这个时候的状态是不确定的。
     * 
     */
    RUNNABLE,

    /**
     * 等待监视锁，这个时候线程被操作系统挂起。当进入synchronized块/方法或者在调用wait()被唤醒/超时之后重新进入synchronized块/方法，
	 * 锁被其它线程占有，这个时候被操作系统挂起，状态为阻塞状态。
     * 
     * 阻塞状态的线程，即使调用interrupt()方法也不会改变其状态。
     * 
     */
    BLOCKED,

    /**
     * 无条件等待，当线程调用wait()/join()/LockSupport.park()不加超时时间的方法之后所处的状态，如果没有被唤醒或等待的线程没有结束，
	 * 那么将一直等待，当前状态的线程不会被分配CPU资源和持有锁。
     * 
     */
    WAITING,

    /**
     * 有条件的等待，当线程调用sleep(睡眠时间)/wait(等待时间)/join(等待时间)/ LockSupport.parkNanos(等待时间)/LockSupport.parkUntil(等待时间)
	 * 方法之后所处的状态，在指定的时间没有被唤醒或者等待线程没有结束，会被系统自动唤醒，正常退出。
     * 
     */
    TIMED_WAITING,

    /**
	 * 终止线程的线程状态。
	 *
	 * 执行完了run()方法。线程已完成执行；其实这只是Java语言级别的一种状态，在操作系统内部可能已经注销了相应的线程，
	 * 或者将它复用给其他需要使用线程的请求，而在Java语言级别只是通过Java代码看到的线程状态而已。
     * 
     */
    TERMINATED;
}
```

#### 各种状态之间切换图

![image02](/images/2019-09-10-Multiy_Thread_Basis/image02.png)

#### 状态切换代码演示

**NEW--->RUNNABLE**
```java
public class MyThread extends Thread {
    private String threadName;

    private MyThread(String threadName) {
        this.threadName = threadName;
    }

    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        System.out.println("调用线程run方法-->开始");
        try {
            //业务逻辑处理
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            // 主动中断线程
            Thread.currentThread().interrupt();
        }
        System.out.println("调用线程run方法-->结束");

    }

    public static void main(String[] args) {
        MyThread myThread01 = new MyThread("myThread01");
        System.out.println(myThread01.getThreadName() + "调用start()之前 state is : " + myThread01.getState());
        myThread01.start();
        System.out.println(myThread01.getThreadName() + "调用start()之后 state is : " + myThread01.getState());
    }
}
```

**RUNNABLE--->BLOCKED**
```java
public class MyThread extends Thread {
    private String threadName;
    private final Object objectLock;

    private MyThread(String threadName, Object objectLock) {
        this.threadName = threadName;
        this.objectLock = objectLock;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        System.out.println("调用线程run方法-->开始");
        synchronized (this.objectLock) {
            System.out.println(this.threadName+"获取synchronized锁");
            try {
                //业务逻辑处理
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                // 主动中断线程
                Thread.currentThread().interrupt();
            }
        }
        System.out.println("调用线程run方法-->结束");

    }

    public static void main(String[] args) throws InterruptedException {
        Object objectLock =new Object();
        MyThread myThread01 = new MyThread("myThread01",objectLock);
        MyThread myThread02 = new MyThread("myThread02",objectLock);
        myThread01.start();
        myThread02.start();
        System.out.println(myThread01.getThreadName() + "调用start()之后 state is : " + myThread01.getState());
        Thread.sleep(100);
        System.out.println(myThread02.getThreadName() + "调用start()之后 state is : " + myThread02.getState());
    }
}
```

**RUNNABLE--->WAITING/TIMED_WAITING**
```java
public class MyThread extends Thread {
    private String threadName;
    private final Object objectLock;
    private String type;
    private List<String> products = new ArrayList<>();

    private MyThread(String threadName, Object objectLock, String type) {
        this.threadName = threadName;
        this.objectLock = objectLock;
        this.type = type;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        System.out.println(this.threadName + "调用线程run方法-->开始");
        synchronized (this.objectLock) {
            System.out.println(this.threadName + "获取objectLock的synchronized锁");
            //生产者
            if ("Producer".equals(this.type)) {
                this.products.add(this.threadName);
                System.out.println(this.threadName + "生产出一个物品，通知消费者消费");
                this.objectLock.notify();
                System.out.println(this.threadName + "成功通知了消费者");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    // 主动中断线程
                    Thread.currentThread().interrupt();
                }
            }
            //消费者
            if ("Consumer".equals(this.type)) {
                try {
                    System.out.println(this.threadName + "检查不存在物品，等待生产者生产");
                    this.objectLock.wait();
                    System.out.println(this.threadName + "收到消费了一个物品");
                } catch (InterruptedException e) {
                    // 主动中断线程
                    Thread.currentThread().interrupt();
                }
            }
        }
        System.out.println(this.threadName + "调用线程run方法-->结束");

    }

    public static void main(String[] args) throws InterruptedException {
        Object objectLock = new Object();
        MyThread myThread01 = new MyThread("myThread01", objectLock, "Consumer");
        MyThread myThread02 = new MyThread("myThread02", objectLock, "Producer");
        myThread01.start();
        System.out.println(myThread01.getThreadName() + "调用start()之后 state is : " + myThread01.getState());
        Thread.sleep(100);
        System.out.println(myThread01.getThreadName() + "调用objectLock.wait()之后 state is : " + myThread01.getState());
        myThread02.start();
        System.out.println(myThread02.getThreadName() + "调用start()之后 state is : " + myThread02.getState());
        Thread.sleep(100);
        System.out.println(myThread02.getThreadName() + "调用objectLock.notify()之后 state is : " + myThread02.getState());
    }
}
```
`注意：这是一个简单的生产者消费者模式；在这里wait/notify必须写在synchronized 保护的代码块中，在synchronized使用wait方法，而在执行 wait 方法之前，必须先持有对象的 monitor 锁，也就是通常所说的 synchronized 锁。原因就是为了保证代码先进行wait再进行notify；否则会发生死等`


#### 各种状态切换方法说明

##### wait/notify/notifyAll

wait方法分两种，一种无参的`wait()`一种有参的`wait(timeout)`；一种是让线程进入`WAITING`；另一种是让线程进入`TIMED_WAITING`；带参的是让线程进入一个有效期的等待，超过有效期则会自动唤醒；无参的会让线程进入一个死等的状态。

这些方法只能由同一对象锁的持有者线程调用，也就是写在同步块里面，否则会抛出IllegalMonitorStateException异常。
   * wait方法导致当前线程等待，加入该对象的等待集合中，并且放弃当前持有的对象锁。
   *  notify/notifyAll方法唤醒一个或所有正在等待这个对象锁的线程。
   **注意:** 虽然会wait自动解锁，但是对顺序有要求，如果在notify被调用之后，才开始wait方法的调用，则线程会永远处于waiting状态。
**深入了解：**
* 每个对象都可以被认为是一个“监视锁monitor”，这个监视器由三部分组成（一个独占锁，一个入口队列，一个等待队列）。
*  注意是一个对象只能有一个独占锁，但是任意线程都可以拥有这个独占锁。对于对象非同步而言，任意时刻可以有任意个线程调用该方法。（即普通方法同一时刻可以有多个线程调用）。对于对象而言，只有拥有这个对象的独占锁才能调用这个同步方法。
* 如果这个独占锁被其他线程占用，那么另外一个调用该同步方法的线程就会处于阻塞状态，此线程进入入口队列（锁池）。
* 若一个拥有该独占锁的线程调用该对象同步方法的wait()方法，则线程会释放独占锁，并且进入该线程的等待队列（等待池）。
* 某个线程调用notify(),notifyAll()方法是将等待队列的线程转移到入口队列，然后让他们竞争对象的独占锁，因为这个调用线程本身必须拥有锁。

**notify和notifyAll的区别**

notify方法只唤醒一个等待（对象的）线程并使该线程开始执行。所以如果有多个线程等待一个对象，这个方法只会唤醒其中一个线程，选择哪个线程取决于操作系统的实现。如果当前情况下有多个线程需要被唤醒，推荐使用notifyAll方法。比如在生产者-消费者里面的使用，每次都需要唤醒所有的消费者或者生产者，以判断程序是否可以继续往下执行。

notify的例子前面已经给过一个简单的生产者消费者模式；notifyAll的例子其实就是多增加一个消费者线程继续持有对象锁。然后进行全部唤醒
```java
public class MyThread extends Thread {
    private String threadName;
    private final Object objectLock;
    private String type;
    private List<String> products = new ArrayList<>();

    private MyThread(String threadName, Object objectLock, String type) {
        this.threadName = threadName;
        this.objectLock = objectLock;
        this.type = type;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        System.out.println(this.threadName + "调用线程run方法-->开始");
        synchronized (this.objectLock) {
            System.out.println(this.threadName + "获取objectLock的synchronized锁");
            //生产者
            if ("Producer".equals(this.type)) {
                this.products.add(this.threadName);
                System.out.println(this.threadName + "生产出一个物品，通知消费者消费");
                this.objectLock.notifyAll();
                System.out.println(this.threadName + "成功通知了消费者");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    // 主动中断线程
                    Thread.currentThread().interrupt();
                }
            }
            //消费者
            if ("Consumer".equals(this.type)) {
                try {
                    System.out.println(this.threadName + "检查不存在物品，等待生产者生产");
                    this.objectLock.wait();
                    System.out.println(this.threadName + "收到消费了一个物品");
                } catch (InterruptedException e) {
                    // 主动中断线程
                    Thread.currentThread().interrupt();
                }
            }
        }
        System.out.println(this.threadName + "调用线程run方法-->结束");

    }

    public static void main(String[] args) throws InterruptedException {
        Object objectLock = new Object();
        MyThread myThread01 = new MyThread("myThread01", objectLock, "Consumer");
        MyThread myThread03 = new MyThread("myThread03", objectLock, "Consumer");

        MyThread myThread02 = new MyThread("myThread02", objectLock, "Producer");
        myThread01.start();
        myThread03.start();
        System.out.println(myThread01.getThreadName() + "调用start()之后 state is : " + myThread01.getState());
        System.out.println(myThread03.getThreadName() + "调用start()之后 state is : " + myThread03.getState());
        Thread.sleep(100);
        System.out.println(myThread01.getThreadName() + "调用objectLock.wait()之后 state is : " + myThread01.getState());
        System.out.println(myThread03.getThreadName() + "调用objectLock.wait()之后 state is : " + myThread03.getState());
        myThread02.start();
        System.out.println(myThread02.getThreadName() + "调用start()之后 state is : " + myThread02.getState());
        Thread.sleep(100);
        System.out.println(myThread02.getThreadName() + "调用objectLock.notify()之后 state is : " + myThread02.getState());
    }
}
```
**wait，notify来实现等待唤醒功能有两个缺点**

* 1.由上面的例子可知,wait和notify都是Object中的方法,在调用这两个方法前必须先获得锁对象，这限制了其使用场合:只能在同步代码块中。
* 2.另一个缺点可能上面的例子不太明显，当对象的等待队列中有多个线程时，notify只能随机选择一个线程唤醒，无法唤醒指定的线程。

##### 为什么 wait/notify/notifyAll 被定义在 Object 类中，而 sleep 定义在 Thread 类中？

主要有两点原因：

* 1、Java 中每个对象都有一把称之为 monitor 监视器的锁，由于每个对象都可以上锁，这就要求在对象头中有一个用来保存锁信息的位置。这个锁是对象级别的，而非线程级别的，wait/notify/notifyAll 也都是锁级别的操作，它们的锁属于对象，所以把它们定义在 Object 类中是最合适，因为 Object 类是所有对象的父类。
* 2、如果把 wait/notify/notifyAll 方法定义在 Thread 类中，会带来很大的局限性，比如一个线程可能持有多把锁，以便实现相互配合的复杂逻辑，假设此时 wait 方法定义在 Thread 类中，如何实现让一个线程持有多把锁呢？又如何明确线程等待的是哪把锁呢？既然我们是让当前线程去等待某个对象的锁，自然应该通过操作对象来实现，而不是操作线程。

##### wait 和 sleep 方法的异同？

**相同点**
* 1.它们都可以让线程阻塞。
* 2.它们都可以响应 interrupt 中断：在等待的过程中如果收到中断信号，都可以进行响应，并抛出 InterruptedException 异常。

**不同点**
* 1.wait 方法必须在 synchronized 保护的代码中使用，而 sleep 方法并没有这个要求。
* 2.在同步代码中执行 sleep 方法时，并不会释放 monitor 锁，但执行 wait 方法时会主动释放 monitor 锁。
* 3.sleep 方法中会要求必须定义一个时间，时间到期后会主动恢复，而对于没有参数的 wait 方法而言，意味着永久等待，直到被中断或被唤醒才能恢复，它并不会主动恢复。
* 4.wait/notify 是 Object 类的方法，而 sleep 是 Thread 类的方法。

##### LockSupport

**park()/unpark(thread)**
park()阻塞当前线程；
unpark(thread)解锁指定thred线程；
```java
public class MyThread extends Thread {
    private String threadName;

    private MyThread(String threadName) {
        this.threadName = threadName;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        System.out.println(this.threadName + "线程阻塞-->开始");

        LockSupport.park();

        System.out.println(this.threadName + "线程阻塞-->结束");

    }

    public static void main(String[] args) throws InterruptedException {
        MyThread myThread01 = new MyThread("myThread01");
        myThread01.start();
        Thread.sleep(100);
        System.out.println(myThread01.getThreadName() + "线程唤醒-->开始");
        LockSupport.unpark(myThread01);
        System.out.println(myThread01.getThreadName() + "线程唤醒-->结束");

    }
}
// 程序输出
myThread01线程阻塞-->开始
myThread01线程唤醒-->开始
myThread01线程唤醒-->结束
myThread01线程阻塞-->结束
```
下面我们看一下如果唤醒unpark先于阻塞park的情况
```java
public class MyThread extends Thread {
    private String threadName;

    private MyThread(String threadName) {
        this.threadName = threadName;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println(this.threadName + "线程阻塞-->开始");

        LockSupport.park();

        System.out.println(this.threadName + "线程阻塞-->结束");

    }

    public static void main(String[] args) throws InterruptedException {
        MyThread myThread01 = new MyThread("myThread01");
        myThread01.start();
        Thread.sleep(100);
        System.out.println(myThread01.getThreadName() + "线程唤醒-->开始");
        LockSupport.unpark(myThread01);
        System.out.println(myThread01.getThreadName() + "线程唤醒-->结束");

    }
}
//程序输出
myThread01线程唤醒-->开始
myThread01线程唤醒-->结束
myThread01线程阻塞-->开始
myThread01线程阻塞-->结束
```
我们可以看出先唤醒指定线程,然后阻塞该线程，线程并没有真正被阻塞并且可以正常执行完后退出。

`我们下面我们看一下如果唤醒unpark两次都先于阻塞park两次的情况`
```java
public class MyThread extends Thread {
    private String threadName;

    private MyThread(String threadName) {
        this.threadName = threadName;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        for(int i=0;i<2;i++) {
            System.out.println(this.threadName + "线程阻塞-->开始");

            LockSupport.park();

            System.out.println(this.threadName + "线程阻塞-->结束");
        }

    }

    public static void main(String[] args) throws InterruptedException {
        MyThread myThread01 = new MyThread("myThread01");
        myThread01.start();
        Thread.sleep(100);
        for(int i=0;i<2;i++) {
            System.out.println(myThread01.getThreadName() + "线程唤醒-->开始");
            LockSupport.unpark(myThread01);
            System.out.println(myThread01.getThreadName() + "线程唤醒-->结束");
        }

    }
}
//程序输出
myThread01线程唤醒-->开始
myThread01线程唤醒-->结束
myThread01线程唤醒-->开始
myThread01线程唤醒-->结束
myThread01线程阻塞-->开始
myThread01线程阻塞-->结束
myThread01线程阻塞-->开始
并且程序一直处于阻塞状态
```
>>由此我们可以看出：先唤醒线程，在阻塞线程，线程不会真的阻塞；但是先唤醒线程两次再阻塞两次时就会导致线程真的阻塞。

**park()/unpark(thread)线程阻塞唤醒机制**

LockSupport是通过控制一个变量`_counter`来对线程阻塞唤醒进行控制的。
* park()阻塞线程会消耗凭证，此时会直接将`_counter`设置为0；unpark(thread)唤醒线程会获取一个凭证，此时会将`_counter`设置为1；记住这个凭证最多只会有1个，就是不管调用几次park()/unpark(thread)这个凭证要么是1，要么是0；
* 当调用park()进行阻塞线程的时候，此时会直接将`_counter`设置为0；即使连续调用两次park()，`_counter`依旧是0；同时会记录设置为0之前的值，前值小于1(也就是0)的时候就会进行线程阻塞；否则直接退出不进行阻塞。
* 当调用unpark(thread)进行唤醒线程的时候，此时此时会直接将`_counter`设置为1；即使连续调用两次unpark(thread)，`_counter`依旧是1；同时会记录设置为1之前的值，前值小于1(也就是0)的时候就会进行线程唤醒；否则直接退出不进行唤醒。

所以就可以得出：
`为什么可以先唤醒线程后阻塞线程？`
>>因为：先unpark(thread)唤醒线程，`_counter=1`；当park()阻塞线程，_counter_old>=1则不进行阻塞，直接退出；
`为什么唤醒两次后阻塞两次会阻塞线程？`
>>因为：先连续unpark(thread)唤醒线程两次(不管多少次)，`_counter=1`；后面连续两次park()阻塞线程，第一次判断_counter_old>=1则不进行阻塞，直接退出；后面会直接设置`_counter=0`；第二次判断_counter_old>=1就不满足，所以会进行线程阻塞。

**`总结：LockSupport是JDK中用来实现线程阻塞和唤醒的工具。使用它可以在任何场合使线程阻塞，可以指定任何线程进行唤醒，并且不用担心阻塞和唤醒操作的顺序，但要注意连续多次唤醒的效果和一次唤醒是一样的。LockSupport真实的实现均在 unsafe`**

**parkNanos(nanos)/parkUntil(deadline)**
* **parkNanos(long nanos)** : 阻塞当前线程，最长不超过nanos纳秒，返回条件在park()上增加了超时时间
* **parkUntil(long deadline)** : 阻塞当前线程，直到deadline时间

**在Java6中，LockSupport增加了3个方法：**
```java
public static void park(Object blocker)
public static void parkNanos(Object blocker, long nanos)
public static void parkUntil(Object blocker, long deadline)
参数：
			blocker：用来标识当前线程正在等待的对象(即阻塞对象)，主要用于排查问题和系统监控。
	
		在线程dump时，带blocker参数的park()方法比不带blocker参数的park()方法多出以下内容(即指出了阻塞对象的类型)：
			- parking to wait for <0x000...> （ com.jxn.test.TestLockSupport）
			
		说明：
			1)当线程(因使用synchronized关键字)阻塞在一个对象上时，通过线程dump能够查看到该线程的阻塞对象，从而可以方便地定位问题。
			2)使用LockSupport中不带blocker参数的park()方法来阻塞对象时，通过线程dump无法看到该线程的阻塞对象；故在java6中，提供了带blocker参数的park()方法来解决这一问题。
```

##### join

join方法是实现线程同步，可以将原本并行执行的多线程方法变成串行执行的；

join方法使用在线程start方法之后。放在start方法之前基本无作用，但是不会报错；

join也有两个方法，一个是无参的`join()`另一个是有参的`join(millis)`；无参的表示线程使用join方法之后在执行完成后才执行下一个线程。有参的表示线程使用join方法之后在执行指定时间之后才执行下一个线程。如果线程在指定时间之内执行完，则线程执行完毕之后继续下一个线程无需等待指定时间；如果线程在指定时间之内未执行完，当时间到期时就执行下一个线程。
**示例**
```java
public class MyThread extends Thread {
    private String threadName;

    private MyThread(String threadName) {
        this.threadName = threadName;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        System.out.println(this.threadName + "调用线程run方法-->开始");

        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            // 主动中断线程
            Thread.currentThread().interrupt();
        }

        System.out.println(this.threadName + "调用线程run方法-->结束");

    }

    public static void main(String[] args) throws InterruptedException {
        MyThread myThread01 = new MyThread("myThread01");
        MyThread myThread02 = new MyThread("myThread02");
        myThread01.start();
        myThread01.join();
        myThread02.start();
        System.out.println(myThread01.getThreadName() + "调用join()之后 state is : " + myThread01.getState());
        System.out.println(myThread02.getThreadName() + "调用start()之后 state is : " + myThread02.getState());

    }
}
```

##### interrupt

interrupt()方法: 作用是中断线程。将会设置该线程的中断状态位，即设置为true，中断的结果线程是死亡、还是等待新的任务或是继续运行至下一步，就取决于这个程序本身。线程会不时地检测这个中断标示位，以判断线程是否应该被中断（中断标示值是否为true）。它并不像stop方法那样会中断一个正在运行的线程　

interrupt()方法只是改变中断状态，不会中断一个正在运行的线程。需要用户自己去监视线程的状态为并做处理。支持线程中断的方法（也就是线程中断后会抛出interruptedException的方法）就是在监视线程的中断状态，一旦线程的中断状态被置为“中断状态”，就会抛出中断异常。这一方法实际完成的是，给受阻塞的线程发出一个中断信号，这样受阻线程检查到中断标识，就得以退出阻塞的状态。 

更确切的说，如果线程被Object.wait, Thread.join和Thread.sleep三种方法之一阻塞，此时调用该线程的interrupt()方法，那么该线程将抛出一个 InterruptedException中断异常（该线程必须事先预备好处理此异常），从而提早地终结被阻塞状态。如果线程没有被阻塞，这时调用 interrupt()将不起作用，直到执行到wait(),sleep(),join()时,才马上会抛出 InterruptedException。也可以通过thread.isInterrupted()来获取线程的中断状态。
**示例**
```java
public class MyThread extends Thread {
    private String threadName;

    private MyThread(String threadName) {
        this.threadName = threadName;
    }


    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        System.out.println(this.threadName + "调用线程run方法-->开始");
        int i = 0;
        while (true) {
            i++;
            if (i > 100000000) {
                break;
            }
        }
        try {
            System.out.println(this.threadName + " i=" + i);
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            // 主动中断线程
            System.out.println(this.threadName + " 线程中断");
            Thread.currentThread().interrupt();
        }

        System.out.println(this.threadName + "调用线程run方法-->结束");

    }

    public static void main(String[] args) {
        MyThread myThread01 = new MyThread("myThread01");
        myThread01.start();
        myThread01.interrupt();
        System.out.println(myThread01.getThreadName() + "调用interrupt()之后 state is : " + myThread01.getState());
    }
}
```

##### yield()

```java
public static native void yield();
```
这个方法是一个静态本地方法，是“让步”的意思，主要是让出CPU资源；一旦执行，它会使当前线程让出CPU，即由“运行状态”进入到“就绪状态”，从而让其它具有相同优先级的等待线程获取执行权。但是要注意的是让出CPU并不代表当前线程不执行了。当前线程让出CPU后，还是会进行CPU资源的争夺，但是能不能再次被分配到，就不一定了。因此yeild()方法的调用好像就是在说：我已经完成一些最终要的工作了，应该可以休息一下了，可以给其他线程一些工作机会了！

如果你觉得一个线程不那么重要，或者优先级比较低，但是又不想它占用过多的CPU资源，就可以在适当的时候使用Thread.yield()，给予其他重要线程更多的机会。

### 守护线程

java中线程主要分为两大类：一种是用户线程(这个是我们平常开发中用到的)；另一个就是守护线程，守护线程我们可以简单的认为就是为了守护用户线程的，当然功能远远不止于此；JVM最典型的守护线程就是GC(垃圾回收器)

在Thread类里面提供有如下的守护线程的操作方法：
* 设置为守护线程：public final void setDaemon(boolean on);
* 判断是否为守护线程：public final boolean isDaemon();

`注意：`
`1、thread.setDaemon(true)必须在线程启动之前，即thread.start()方法之前设置，否则会跑出一个IllegalThreadStateException异常。因为正在运行的线程我们不能改变它的性质`
`2、在守护线程中新建其它线程也是守护线程`
`3、守护线程会随着所有用户线程的用户线程的结束而结束；用户线程结束那么JVM也会停止工作，即使守护线程正在工作也会直接结束，所以有些重要的必须完成的逻辑处理不应该放入守护线程`

**示例**
```java
public class MyThread extends Thread {
    private String threadName;

    private MyThread(String threadName) {
        this.threadName = threadName;
    }

    public static void main(String[] args) {
        MyThread myThread01 = new MyThread("myThread01");
        myThread01.setDaemon(true);
        myThread01.start();
        System.out.println("用户线程结束");

    }

    private String getThreadName() {
        return threadName;
    }

    @Override
    public void run() {
        try {
            //守护线程阻塞2秒后运行
            Thread.sleep(2000);
            File f = new File("C:/Users/yeming.gao/Desktop/test.txt");
            FileOutputStream os = new FileOutputStream(f, true);
            os.write("test".getBytes());
            os.close();
            System.out.println(this.threadName + "守护线程所有业务处理结束");
        } catch (IOException | InterruptedException e1) {
            Thread.currentThread().interrupt();
        }
    }
}
//当我们没有设置为守护线程的时候程序运行的结果为：（并且指定路径会出现test.txt文件说明myThread01线程执行完成）
用户线程结束
myThread01守护线程所有业务处理结束
//当我们设置为守护线程程序运行结果为：（并且指定路径没有出现文件，说明myThread01没有执行完成就已经结束）
用户线程结束
```

所以我们可以理解为守护线程的目的是为了守护用户线程，当用户线程结束之后，守护线程就没有存在的必要，所以守护线程也就可以直接结束；虽然守护线程实际的应用不多，但是针对自己的业务情况或者做一些监控什么的还是可以用得到的。

### Java的内存模型

![image03](/images/2019-09-10-Multiy_Thread_Basis/image03.png)

**Java内存模型-操作规范：**
* 1，共享变量必须存放在主内存中
* 2，线程有自己的工作内存，线程只可操作自己的工作内存
* 3，线程要操作共享变量，需从主内存中读取到工作内存，改变值后需要从工作内存同步到主内存中。

`（当涉及共享变量的时候这个时候就会引出线程安全的问题）`

**Java内存模型-同步交互协议，规定了8种原子操作：**
* 1，lock(锁定)：将主内存中的变量锁定，为一个线程所独占
* 2，unlock(解锁)：将lock加的锁定解除，此时其它的线程可以有机会访问此变量
* 3，read(读取)：作用于主内存变量，将主内存中的变量值读到工作内存中
* 4，load(载入)：作用于工作内存变量，将read读取的值保存到工作内存中的变量副本中
* 5，use(使用)：作用于工作内存变量，将值传递给线程的代码执行引擎
* 6，assign(赋值)：作用于工作内存变量，将执行引擎处理返回的值重新赋值给变量副本
* 7，store(存储)：作用于工作内存变量，将变量副本的值传送到主内存中
* 8，write(写入)：作用于主内存变量，将store传送过来的值写入到主内存的共享变量中
利用锁的机制=来实现同步的（解决数据的不一致性 JMM）

#### 锁机制有如下两种特性

* 互斥性：即在同一时间只允许一个线程持有某个对象锁，通过这种特性来实现多线程中的协调机制，这样在同一时间只有一个线程对需要同步的代码块（复合操作）进行访问。互斥性我们也往往称为操作的原子性。
* 可见性：必须确保在锁被释放之前，对共享变量所做的修改，对于随后获得该锁的另一个线程是可见的（即在获得锁时应获得最新共享变量的值），否则另一个线程可能是在本地缓存的某个副本上继续操作从而引起的不一致
（偏向锁--->自旋锁--->轻量级锁--->重量级锁）

#### 锁优化方法
* 1，减少锁持有时间
* 2，减小锁粒度
* 3，锁分离
* 4，锁粗化
* 5，锁消除

`锁这块也只是先了解这些概念，后续再学习更新`