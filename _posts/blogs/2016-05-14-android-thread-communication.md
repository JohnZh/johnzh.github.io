---
layout: post_no_cmt
title:  "Android 线程间通信：Handler/Message/MessageQueue/Looper"
date:   2016-05-14 00:06:00 +0800
categories: [android]
---
Last modified: 2020-04-27

# 举例
```
// UIThread
mHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                mTextView.setText(String.valueOf(msg.obj));
            }
        };

Customer customer = new Customer(mHandler);
customer.start();
Producer producer = new Producer(customer);
producer.start();

// Customer
class Customer extends Thread {
    private Handler mCustomerHandler;
    private Handler mMainHandler;

    public Customer(Handler mainHandler) {
        mMainHandler = mainHandler;
    }

    @Override
    public void run() {
        Looper.prepare();
        synchronized (this) {
            mCustomerHandler = new Handler() {
                @Override
                public void handleMessage(Message msg) {
                    int what = msg.what;
                    StringBuilder builder = new StringBuilder()
                            .append("Get ").append(what).append(" from Producer");
                    Message message = mMainHandler.obtainMessage(0);
                    message.obj = builder.toString();
                    message.sendToTarget();
                }
            };
            notify();
        }
        Looper.loop();
        notify();
    }
}

// Producer
class Producer extends Thread {

    private Customer mCustomer;

    public Producer(Customer customer) {
        mCustomer = customer;
    }

    @Override
    public void run() {
        synchronized (mCustomer) {
            if (mCustomer.mCustomerHandler == null) {
                try {
                    mCustomer.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

        for (int i = 0; i <= 20; i++) {
            mCustomer.mCustomerHandler.sendEmptyMessageDelayed(i, 1000 * i);
        }
    }
}
```
进行子线程到子线程的通信，子线程到 UI 线程的通信，即

Producer -> Customer -> UI Thread

# 其他
Android 中涉及到进程间通信，比如 runOnUiThread(runnable)，View.post(runnable)，AsyncTask 等，其实底层都是使用了 Handler，AsyncTask 则是 Thread 和 Handler 的封装

# 深入
## Handler 

```
// Handler.java
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue; // MessageQueue
    mCallback = callback;
    mAsynchronous = async;
}
```
Handler 核心的构造函数：
- 除了初始化他的成员变量，这里还说明了会发生内存泄露的情况：如果这个 Handler 的子类是匿名内部类 AnonymousClass，或者声明过的内部类 MemberClass，或者局部内部类 LocalClass。
    - 原因就是上述情况下，这些内部类都会持有外部类的引用，当外部类生命周期结束，比如 Activity 关闭，但是 Handler 子类生命周期未结束，还依然持有 Activity 的引用，那么就发生了内存泄露
    - Handler 生命周期未结束主要是由于消息队列里面的消息未全部处理完导致的
- 成员变量的初始化
    - mLooper:Looper，循环，用于无限循环消息队列 (消费消息)
    - mQueue:MessageQueue，这其实是 Looper 里面的消息队列
    - mCallback:Handler.Callback 消息处理完后的回调
    - mAsynchronous:boolean 标记该 Handler 发送的消息是否为异步

## Loop

```
// Handler.java
mLooper = Looper.myLooper();

// Looper.java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}

public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
- Looper#prepare 本质上是为当前线程新建一个 Looper
- Looper#myLooper 则是获取当前线程的 Looper
- 线程中如果没有进行 Looper#prepare 就 new Handler() 会丢出异常，原因是因为没有 Looper 就没有消息队列 MessageQueue，Handler 也就无法 sendMessage()
- UI线程在刚刚启动的时候，ActivityThread#main 里面也调用了 Looper#prepareMainLooper 和 Looper#loop

> ThreadLocal 参考[《ThreadLocal 原理》]({% post_url/blogs/2016-05-16-java-threadlocal %})

## Handler & Message
不论 `Handler#post*` 还是 `Handler#send*Message*` 最后都会走 `Handler#sendMessageAtTime`

```
// Handler.java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
显然最后都是进行了 MessageQueue 的入队操作

### Message 的获取
不论 `Handler#post*` 还是 `Handler#send*Message*` 还是 `Handler#obtainMessage` 最后都会使用到 `Message#obtain(Handler h)` 和 `Message#obtain()`

obtain(Handler h) 是做了 obtain() 之后做做了 msg.handler = h 的操作

```
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}

void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```
Message 使用了对象池机制，**优化内存使用**，对象池结构上采用了单链表的结构：
    
- 获取时候取链表头一个，没有则新建
- 使用完，回收(归还)时，放回链表头

## MessageQueue & Message

```
// MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
- 由于经历 `Hanler#send*` --> `Handler#sendMessageAtTime` --> `Handler#enqueueMessage` --> `MessageQueue#enqueueMessage`，显然由于 `Hanler#send*` 涉及到多线程，`MessageQueue#enqueueMessage` 必然涉及到线程安全，synchronized 包裹了消息入队操作过程
- MQ 在 Quitting 一般是由于 Looper#quit，消息回收，不入队
- 消息标记在用，设置 when (当前时间或者当前时间+延时)，指针指向消息队列头部
    - 这里的消息队列用的还是 Message 的数据结构，链表
- 消息的插入是按照 when 时间序的

## Looper & MessageQueue

```
// Looper.java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}

// MessageQueue.java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit(); // 原生层消息队列的头指针
}
```
显然 Looper 持有一个 MessageQueue，并在调用了 Looper#loop 后开始对 MQ 的消息的消费。下为核心代码：

```
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    ......
    
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        ......
        
        try {
            msg.target.dispatchMessage(msg);
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        
        ......
        
        msg.recycleUnchecked();
    }
}
```
- 死循环，MQ#next 会使线程进入阻塞，等待消息入队后激活，继续循环
- 使用 msg.target:Handler 分发消息：
    - 如果 Message#obtain 时候设置过 Callback 则执行
    - 不然，检查 Handler.Callback 有没有，有就执行 Handler.Callback#handleMessage(msg)
    - Callback 都没有，执行 Handler#handleMessage(msg) 该方法可重载
- 消息回收(放回消息对象池)

# 总结
## 消息流向

```
                  MainThread
                      |
                    Looper ---------+
                      |           loop
           +----> MessageQueue <----+
           |                 | 
       send msg         dispatch msg
           |                 |
           +------ Handler <-+
                      ^  |
                      |  +-- msg -> Handler#handleMessage(msg) 
                      |  
        Thread 1 --- call --- Thread N
                      |
                  Thread 2  
```

## 主线程阻塞和 ANR
问题为：UI 线程底层也有 Looper#loop，内部为死循环，那死循环不会造成 ANR 吗？

ANR 是 Android Not Response，意为线程在处理消息时由于消息太多或者消息处理太久而超时的问题，影响用户体验，App 流畅度，和线程阻塞完全是两件事情。基于事件驱动的 Android 系统，无消息时候的阻塞不会造成 ANR

## 再深入
难点在于 MQ 的阻塞和唤醒，涉及到 native 层，先不展开
- 具体方法为 MQ#next() -> nativePollOnce(ptr, nextPollTimeoutMillis)  该方法会阻塞
- 以及 MQ#enqueueMessage -> nativeWake(mPtr)