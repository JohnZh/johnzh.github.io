---
layout: post_no_cmt
title:  "Java 并发：线程协作 wait/notify/sleep/yield/join"
date:   2016-05-12 18:00:00 +0800
categories: [java]
---
Last modified: 2020-09-13

# 线程协作方法
- `Object#wait`: 等待某个条件而使当前线程发生阻塞，条件由另外线程提供，释放对象锁，必须与 synchronized 一起使用（使用前提是必须 "拥有" 对象锁）
- `Object#notify`, `Object#notifyAll`: 通知由于 wait 阻塞的特定或者全部线程条件已达成，注意，做两个方法**不会主动释放锁**，这意味着调用 wait 的线程要继续执行还需要等获取到锁后才会继续执行
- Thread#sleep: 线程休眠阻塞固定时间，让出 cpu，不释放对象锁
- Thread#yield: 线程主动让出 cpu 给其他线程使用，无固定时间，也不保证马上就让出，因此在少量场景下使用该方法，而主要用于 debug 和 test
- Thread#join: 当前线程等待另外一个线程执行完后再执行

# 举例
生产者消费者模型，生产者 Producer 不停产生商品 Goods，消费者 Customer 不停消费，且中间有一个商店 Store 用于存放商品。

**商店本质上是一个队列，更好的方法是使用 BlockingQueue**

#### wait/notify：

假设有一个生产者和三个消费者：

```
public static void execute() {
    Store store = new Store(new Producer());
    for (int i = 0; i < 3; i++) {
        store.welcome(new Customer(i));
    }
}

static class Goods {
    int id;

    public Goods(int id) {
        this.id = id;
    }

    @Override
    public String toString() {
        return "Goods(" + id + ")";
    }
}

static class Store {
    List<Goods> goodList = new LinkedList<>();
    Producer producer;
    int MAX_GOODS_SIZE = 5;
    ExecutorService service = Executors.newCachedThreadPool();
    boolean closed;


    public Store(Producer producer) {
        this.producer = producer;
        this.producer.store = this;
        this.service.execute(producer);
    }

    public void welcome(Customer customer) {
        customer.store = this;
        service.execute(customer);
    }

    public boolean isClosed() {
        return closed;
    }

    public void close() {
        closed = true;
        service.shutdownNow();
    }

    public void add(Goods goods) {
        goodList.add(goods);
    }

    public boolean isEmpty() {
        return goodList.size() == 0;
    }

    public boolean isFull() {
        return goodList.size() == MAX_GOODS_SIZE;
    }

    @Override
    public String toString() {
        return "Store goods[" + goodList.size() + "]";
    }

    public Goods get() {
        return goodList.remove(0);
    }
}

static class Producer implements Runnable {
    Store store;
    int count;

    @Override
    public void run() {
        while (true) {
            synchronized (store) {
                if (!store.isFull()) {
                    Goods goods = new Goods(count++);
                    store.add(goods);
                    System.out.println("Producer add " + goods + " | " + store);
                    store.notifyAll();
                } else {
                    try {
                        store.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                if (count == 20) {
                    System.out.println("Producer finish today.");
                    store.close();
                    return;
                }
            } // sync

            if ((count & 1) == 0) {
                Thread.yield();
            }
        }
    }
}

static class Customer implements Runnable {
    Store store;
    int id;

    public Customer(int id) {
        this.id = id;
    }

    @Override
    public String toString() {
        return "Customer(" + id + ")";
    }

    @Override
    public void run() {
        while (true) {
            synchronized (store) {
                if (!store.isEmpty()) {
                    Goods goods = store.get();
                    System.out.println(this + " consume " + goods + " | " + store);
                    store.notifyAll();
                } else if (store.isClosed()) {
                    System.out.println("Store closed. " + this + " leave.");
                    return;
                } else {
                    System.out.println(this + " wait.");
                    try {
                        store.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                        return;
                    }
                }
            } // sync
            Thread.yield();
        }
    }
}
```
- 使用 yeild 使线程切换更明显
- 生产者生产完最后一件商品通知完消费者后直接 shutdownNow 线程池
    - shutdownNow 和 shutdown 不同，shutdownNow 会对线程进行 interrupt，而 shutdown 会等待线程全部完成。
    - 消费者没有设置和生产者一样的退出机制。因此只能通过线程池的 interrupt 来退出
- 消费者收到线程池的 interrupt 后
    - 阻塞的会通过异常捕获退出
    - 非阻塞的消费完成后通过下一个循环条件退出

> 对应的显示锁实现为 ReentrantLook$Condition，对应方法为 Condition#await/Condition#signal/Condition#signalAll

#### wiat 补充 
- wait(0) 等于 wait()
- wait(1000)，1000 毫秒后自动被 notify，也仍可以通过 notify 提前唤醒

#### wait 信号量错失
避免像这样的代码：

```
// run():
while(condition) {
    synchronized(obj) {
        obj.wait();
    }
}
```
多线程的情况下，当有线程 a 已经被 wait 阻塞的时候，可能有其他线程 b 处于 **while(condition)** 的位置

此时如果有线程 c 进行了 notify 并改变了 condition 为 false，那么线程 b 会错过 condition 的改变，继续进入 wait 导致的阻塞。同时会对其他想获取锁的线程造成死锁。

合适的写法：

```
// run():
synchronized(obj) {
    while(condition) {
        obj.wait();
    }
}
```

#### Thread#join
当前线程等待另外一个线程执行完后再执行，异步变同步

join 的实现其实是使用了 Object#wait

以下为 Java7 的代码

```
public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```
- join() 等于 join(0)
- 假设当前线程调用了 start 的线程 t.join()，那么调用关系就会是：currentThread.run() -> t.join() -> synchronizd(this) { while (isAlive()) { this.wait() } }

代码就会像这样：
    
```
void run() { // currentThread
    ......
    
    synchronized(t) {
        while(t.isAlive()) {
            t.wait()    
        }
    }
    
    ......
}
```

无疑，currentThread 阻塞，释放了 t 的对象锁，等待其他线程获得 改变条件后再通知 currentThread 继续执行

那么问题来了，t 在 run 执行的过程中并不会必须有 notify 的代码，那 t 在 run 结束时，currentThread 能否从 wait 的阻塞恢复运行呢？

测试代码：

```
// main.java
T1 t1 = new T1();
T2 t2 = new T2(t1);
t1.start();
t2.start();

Thread.sleep(3000);
System.out.println("main end");

static class T1 extends Thread {
    @Override
    public void run() {
        System.out.println("t1 start");

        for (int i = 0; i <= 100000000; i++) {
            if (i == 0 || i == 1000
                    || i == 10000 || i == 100000
                    || i == 1000000 || i == 10000000
                    || i == 100000000) {
                System.out.println("t1 is running: " + i);
            }
        }
        System.out.println("t1 end");
    }
}

static class T2 extends Thread {
    T1 t1;

    public T2(T1 t1) {
        this.t1 = t1;
    }

    @Override
    public void run() {
        System.out.println("t2 start");
        synchronized (t1) {
            try {
                System.out.println("t2 call t1.wait in synchronized");
                t1.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("t2 end");
    }
}
```
目的很明确，就是想知道 t2.run() 里面调用了 t1.wait() 阻塞之后，t1.run() 之后是否会让 t2 恢复运行还是继续阻塞，运行结果：

```
//output:
t1 start
t1 is running: 0
t1 is running: 1000
t2 start
t2 call t1.wait in synchronized
t1 is running: 10000
t1 is running: 100000
t1 is running: 1000000
t1 is running: 10000000
t1 is running: 100000000
t1 end
t2 end
main end
```
t2 恢复了！

**注意：当前无法从 java 层里面找到任何 notify 导致其恢复的代码，只是可以明确，t.join() 调用后，t 如果已经完成，t.join 会返回，也就是上面测试代码的现象。具体实现也许在 native 层**