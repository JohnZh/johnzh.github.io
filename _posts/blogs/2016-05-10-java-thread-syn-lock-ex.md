---
layout: post_no_cmt
title:  "Java 并发：线程，同步，锁，异常捕获"
date:   2016-05-10 00:06:00 +0800
categories: [java]
---
Last modified: 2020-04-14

# 线程
一个线程就是进程中一个单一顺序控制流（子任务）

线程也是并发的基础条件

# 线程状态

```
    +--> 就绪(Runnable) --+
    |       |   ^         |
新建(new)   |   |         |->死亡(Dead) 
    |       v   |         |
    +--> 阻塞(Blocked) ---+
```
- 新建：初始化状态，获得了必要的资源，以及获得 CPU 时间片资格
- 就绪：调度器分配给其时间片，线程就运行了
- 阻塞：相对于就绪的另外一个状态，某个条件阻止了其运行，此状态下调度器会忽略线程，直到重新进入就绪，调度器会继续分配给其时间片
- 死亡：任务结束，一般为 run 方法的返回，正常结束，以及线程中断结束。Java 已经废除了线程强制关闭 stop，取代它的就是中断

# 线程安全

## synchronized 同步
- 两种方式，同步方法块，同步代码块，本质上都是对对象进行加锁（或者叫作“持有对象锁”），当使用 synchronized 同步对象的时候，对象锁计数器加一，离开 synchronized 方法或者块的时候，对象锁计数器减一，零的时候锁完全释放。
- 基础型数据无法使用 synchronized，因为没有对象锁
- 本质上不存在类锁，实际上 synchronized(Clazz.class) 里面，Clazz.class 是返回了 Clazz 这个类的一个 Class<Clazz> 对象，因此也是对象加锁

### 举例说明

```
// Test.java

......

public void f0() {}

public synchronized void f1() {}

public void f2() {
    synchronized(this) {}
}
```
f1 和 f2 都是本质上都是对本对象 this 加锁，当线程 T(1) 访问（调用）f1 或者 f2 的时候，其他线程 T(n) 无法访问f1 和 f2，但是可以访问其未同步区域，例如 f0()

```
public void f3() {
    synchronized(Test.class) {}
}

private Object obj = new Object();

public void f4() {
    synchronized(obj) {}
}

```
f3 和 f4 本质上都是对“非当前对象加锁”，但是 f3 的锁是 Class<Test> 类型的对象的，f4 是 obj 的锁，因此当线程 T(1) 访问 f3 的时候，f0，f1，f2，f4 都可以被其他线程 T(n) 访问。f4 同理，访问 f1 或者 f2 时候也同理

```
public synchronized static void f5() {}
```
静态方法由于属于类的，因此它与 f3 获取一样的锁。也就是说当线程 T(1) 调用 Test.f5() 的时候，f3 无法被其他线程 T(n) 访问，其他方法都可以被访问

### synchronized 方法 vs synchronized(this) 代码块
synchronized(this) 代码块也叫临界区，其实是 synchronized 方法的一种优化。

synchronized 方法将整个方法体加锁，有时可能并非都是临界资源，非临界资源访问不加锁，只加锁临界资源，这无疑是减小了开销。

相同时间下，线程访问次数更多；执行相同的工作下，时间耗时更少。


## synchronized vs 显式 Lock 对象

具体指 java.util.concurent.locks 包里面锁类，例如 ReentrantLock

以 ReentrantLock 为例子说几点特性：

- 使用 lock#luck - lock#unlock 进行加解锁牺牲了优雅而带来的灵活性
- tryLock(), tryLock(timeout, TimeUnit) 尝试加锁，带超时的尝试加锁。**作用：得知加锁失败，放弃阻塞，转而做其他工作**

```
public void f6() {
    boolean tryLock = lock.tryLock();
    if (tryLock) {
        try {
            
        } finally {
            if (tryLock) {
                lock.unlock();
            }
        }
    } else {
        
    }
}
```
线程 T(1) 访问 f6，加锁进入临界资源操作，其他线程 T(n) 同时访问 f6，在加锁失败后可以先去执行 else {} 里面的工作

- 可中断，synchronized 是不可中断的。相关方法 lock#lockInterruptibly，下为该方法的说明

```
除非当前线程被中断，不然获得锁
如果锁未被其他线程持有，当前线程获得锁，并马上返回，锁计时器设为 1
如果当前线程已获得锁，锁计数器增加 1，方法马上放回
如果锁被其他线程持有，那么当前线程会阻塞等待，直到下面两种情况其一发生：
- 锁被当前线程获得
- 其他线程中断了当前线程

如果当前线程
- 其中断信号已经被设置，然后再调用该方法
- 或者被中断的同时调用该方法正在获取锁
那么 InterruptedException 会被抛出，且中断信号会被清空
```
**并不是说该调用该方法加锁后，后续的代码运行可以被中断且抛出 InterruptedException**。抛出 InterruptedException 的情况只会发生在：
- Thread#interrept 调用后，依然进行 lock#lockInterruptibly
- Thread 等待获取锁阻塞，此时调用 Thread#interrept

```
public void f7() {
    while (true) {
        Thread.yield();
        if (Thread.currentThread().isInterrupted()) {
            break;
        }
    }
    try {
        lock.lockInterruptibly();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        if (lock.isLocked()) {
            lock.unlock();
        }
    }
}
```
此为第一种情况，T(1) 访问 f7，由于 yield 方法，线程一直在让出 CPU 。接着，在其他线程，调用 T(1)#interrept，设置中断信号，跳出循环后，lock.lockInterruptibly()，抛出异常，结束

```
public void f8() {
    try {
        lock.lockInterruptibly();
        System.out.println(Thread.currentThread().getName() + " get lock.");
        while (true) {
            Thread.yield();
            if (Thread.currentThread().isInterrupted()) {
                break;
            }
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
        System.out.println(Thread.currentThread().getName() + " interrupted.");
    } finally {
        if (lock.isLocked()) {
            lock.unlock();
            System.out.println(Thread.currentThread().getName() + " unlock and leave.");
        }
    }
}
```
第二种情况，T(1) 和 T(2) 同时访问 f8，T1 先获得锁，且由于 yield 一直在循环让出 cpu，但依然持有锁，T(2) 的访问就处于一个阻塞等锁的状态。接着，其他线程调用 T(2)#interrept，T(2) 抛出异常，结束

> 关于线程中断，参考[《Java：线程中断》]({% post_url/blogs/2016-05-11-java-thread-interrupt %})

# 异常捕获
在不做任何设置的情况下，子线程的异常是无法在主线程里面捕获的，比如 main 里面去捕获 new Thread() 里面的异常。

两个方式：

```
Thread t1 = new Thread();

t1.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
    @Override
    public void uncaughtException(Thread thread, Throwable throwable) {
    }
});

所有线程的异常统一处理
Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
    @Override
    public void uncaughtException(Thread thread, Throwable throwable) {
    }
});
```

线程池使用中还需要使用 "线程工厂" 进行单个线程的构造：

```
static class HandleExceptionThreadFactory implements ThreadFactory {
    @Override
    public Thread newThread(Runnable runnable) {
        Thread thread = new Thread(runnable);
        thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
            @Override
            public void uncaughtException(Thread thread, Throwable throwable) {
            }
        });
        return thread;
    }
}
ExecutorService service
        = Executors.newCachedThreadPool(new HandleExceptionThreadFactory());
```