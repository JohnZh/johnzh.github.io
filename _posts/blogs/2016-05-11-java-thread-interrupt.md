---
layout: post_no_cmt
title:  "Java 并发：线程中断"
date:   2016-05-11 00:06:00 +0800
categories: [java]
---
# 线程中断

## 线程关闭

为什么不用 Thread#stop？

因为不安全，执行 Thread#stop 的时候，不论 run 方法是否执行完成，线程都会释放所有的对象锁，这是非常危险的，极有可能造成数据的不一致

## 替代方案
为了让线程的关闭更加的可控，在线程退出上应该采用线程中断

### 相关方法

1. `Thread#interrupt` 设置当前线程的中断信号，一些情况下会抛出 InterruptedException，并清除中断信号
2. `Thread#isInterrupted` 判断当前线程的中断信号是否设置
3. 静态方法 `Thread#interrupted` 

```
// Thread.java
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}

public boolean isInterrupted() {
    return this.isInterrupted(false);
}
```
方法 3 本质是调用了当前线程的 Thread#isInterrupted，参数 true 的情况会 **清除线程中断状态的信号**。即当前线程已经被设置中断信号，那么调用方法 3 判断完中断状态为 true 之后，期间不发生中断的情况下，再调用一次返回值会变成 false

### 中断举例
失败的中断：
```
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        while (true) {
            Thread.yield();
        }
    }
}, "T1");
t1.start();
t1.interrupt();
```
使用循环和线程 cpu 让出来模拟线程任务的执行和 cpu 的切换，直接调用 `t1.interrupt()` 无法中断，代码会一直运行下去。因为 `t1.interrupt()`只会设置中断信号，并不会中断运行中的线程。正确的中断：

```
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        while (true) {
            Thread.yield();
            if (Thread.currentThread().isInterrupted()) {
                break; // 也可以 return;
            }
        }
    }
}, "T1");
t1.start();
t1.interrupt();
```

## 抛出中断异常 InterruptedException 进行中断

一旦抛出中断异常 InterruptedException，中断信号会清理，即 `Thread.currentThread().isInterrupted() == false`，因此我们需要在异常捕获中**重新设置中断信号**

```
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        while (true) {
            if (Thread.currentThread().isInterrupted()) {
                return;
            }
            try {
                Thread.sleep(4000);
            } catch (InterruptedException e) {
                e.printStackTrace();
                System.out.println("isInterrupted: " + Thread.currentThread().isInterrupted());
                Thread.currentThread().interrupt();
            }
        }
    }
}, "T1");
t1.start();
try {
    Thread.sleep(2000); //主线程设置休眠 2000 ms 是为了让 t1 能运行到 sleep
} catch (InterruptedException e) {
    e.printStackTrace();
}
t1.interrupt();

// output
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.john.purejava.MyCode$1.run(MyCode.java:29)
	at java.lang.Thread.run(Thread.java:748)
isInterrupted: false
```
### 抛出 InterruptedException 的方法
对于一个会抛出 InterruptedException 的方法，一般来说肯定是阻塞方法

什么是阻塞方法？
将线程从就绪状态变成阻塞状态的方法，线程调用该方法后，需要一个外部条件，才能让线程重新恢复到就绪状态。

比如：
- Thread#sleep 阻塞等待休眠时间到达
- Thread#join 当前线程上，对另一个线程对象 t 进行 t#john 的调用，那么当前线程挂起，阻塞等待 t 完成任务
- ReentrantLock#lockInterruptibly 当前线程发现锁被占用，阻塞等待锁
- Object#wait 当前线程挂起，让出对象锁，阻塞，等待别的线程 notify