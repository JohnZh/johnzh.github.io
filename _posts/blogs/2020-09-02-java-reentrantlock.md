---
layout: post_no_cmt
title:  "ReentrantLock 使用与原理"
date:   2020-09-02 00:06:00 +0800
categories: [java]
---

# ReentrantLock 基本使用

```
ReentrantLock lock = new ReentrantLock(true); // false 或无参非公平锁，true 公平锁
lock.lock();  // 获取锁，如果获锁失败，线程进入阻塞（休眠）
lock.tryLock(); // 尝试获取锁，失败不会阻塞
lock.lockInterruptibly(); //获取锁，如果获锁失败，线程进入阻塞，支持获锁阻塞的中断退出
    
lock.unlock(); // 释放锁
```

# ReentrantLock 源码
## 类结构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903162354794.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)

in short：
- AQS（AbstractQueuedSynchronizer）继承 AOS（AbstractOwnableSynchronizer）
- ReentrantLock 内部静态抽象类 Sync 继承 AQS
- ReentrantLock 内部静态 final 类 FairSync 和 NonfairSync 继承 Sync，分别是公平锁和非公平锁的实现
- AQS 内部有 head & tail 节点，代表一个双向链表实现的队列，代表当前获得锁在执行的线程，以及等待的线程

## 流程（公平锁）

```
ReentrantLock.lock -> FairSync.lock -> FairSync.acquire(1)
```

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```
// FairSync.tryAcquire(int acquires)
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) { 
    
    	// 获取当前 state，判断是否可以获取锁
    	// 0 代表无其他线程持有锁，可以获取，非 0 反之
    	// 队列判断：hasQueuedPredecessors() 返回 false 的情况会进行 CAS 加锁
    	// false 情况 1：head 和 tail 指向相同，即空队列和队列只有一个节点
    	// false 情况 2：head 和 tail 指向不同，即队列最少 2 个节点
    	// s = h.next 必不为 null，当前线程为第二个节点的线程的时候
    	// 以上情况，进行 CAS，成功改变 state 为非 0 时，代表获锁成功
    	
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            // 设置当前唯一获锁执行的线程
            setExclusiveOwnerThread(current); 
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { 
    	// 此处为重入锁的处理。 单纯的 state 计数加 1
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
假设 `tryAcquire(arg)` 返回 true 获锁成功，则无需继续执行 `acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`

假设 `tryAcquire(arg)` 返回 true 获锁失败，则继续执行：

```
acquireQueued(addWaiter(Node.EXCLUSIVE), arg)：

private Node addWaiter(Node mode) {
    Node node = new Node(mode); // 创建当前线程节点

    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
	        // 新节点的 prev 指针指向队尾
            U.putObject(node, Node.PREV, oldTail); 

            // CAS 队尾指针指向新节点（比较 tail 是否等于 oldTail，等于就把 tail 指向 node）
            if (compareAndSetTail(oldTail, node)) { 
 				
 				// 队列原队尾的节点的 next 指针指向新节点。新节点成为新队尾
                oldTail.next = node; 
                return node;
            }
        } else {
        	// 初始化队列
            initializeSyncQueue();
        }
    }
}
```
只有一个线程获取锁的情况下，并没有任何队列节点，直到出现第二个线程来竞争锁的时候，这个时候队列的 tail 为 null，初始化队列 `initializeSyncQueue()`

```
private final void initializeSyncQueue() {
    Node h;
    if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
        tail = h;
}
```
初始化队列加入了一个空节点（thread 为 null）来代表之前获取了锁的线程，同时，队列的 head 和 tail 都指向空节点。

由于在 for 循环中，再执行一次，这次会把之前 new 的当前线程节点加入到队尾，同时 return node，退出 for 循环。

执行完 addWaiter(Node.EXCLUSIVE) 加入当前线程节点到队尾后，继续做 `acquireQueued(final Node node, int arg) `

```
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor(); 
            // 获取新加入节点的前一个节点 p
            // 如果新加入节点的前节点为 head（即 thread为 null 的节点），再尝试获取一次锁
            // 做这次操作的原因是因为空节点之后的节点的线程是最右资格尝试获取锁的线程
            // 并且在 addWaiter 操作的这段时间里面可能之前的线程已经释放了锁，故尝试
        	// 由此也可知，位于队列第二个节点之后的节点是不会进行二次尝试获锁的 
        	
            if (p == head && tryAcquire(arg)) {
            	// 如果尝试获锁成功，把新节点设为队头 head
            	// 同时置 null 它的 thread 和 pre（变成了空节点）
            	// 原队头，即空节点 p，移除队列，等待 GC
            	// return 退出循环
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            
			// 如果这次（第二次）尝试加锁依然没成功
			// 判断失败后是否应该 park（shouldParkAfterFailedAcquire() 代码在下方）
			// 首次循环会 p 的 waitStatus 为 0，那此次循环中会把 waitStatus 设为 -1 SIGNAL
			// 并返回 fasle
			// 再第二次循环中，由于 waitStatus 为 SIGNAL 返回 true，执行 parkAndCheckInterrupt()
			// parkAndCheckInterrupt() 会调用  LockSupport.park(this) 
			// 这会调用 Unsafe 的 park()，这里就实现了线程真正的休眠
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted(); // 线程 interrupted 标记位会被清理为 false
}
```

```
ReentrantLock.unlock -> FairSync.release(1)
```

```
public final boolean release(int arg) {
     if (tryRelease(arg)) {
         Node h = head;
         if (h != null && h.waitStatus != 0)
             unparkSuccessor(h);
         return true;
     }
     return false;
 }

protected final boolean tryRelease(int releases) {
     int c = getState() - releases;
     if (Thread.currentThread() != getExclusiveOwnerThread())
         throw new IllegalMonitorStateException();
     boolean free = false;
     if (c == 0) {
         free = true;
         setExclusiveOwnerThread(null);
     }
     setState(c);
     return free;
 }
```
unlock 里面对 state 的操作是减一，这也说明了，重入锁的 lock 和 unlock 必须是成对出现

释放锁成功后，会对队头节点调用 `unparkSuccessor(h)`：

```
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
首先，队头节点的 waitStatus 从 SIGNAL 置为 0。

其次把它后面的节点，即第二个节点进行 unpark 操作。于是，之前被 park 的线程从 `parkAndCheckInterrupt()` 返回并返回了 `Thread.interrupted()` 的值。

线程从 `parkAndCheckInterrupt()` 返回后，假设线程没有被 interrupt，那么它会继续执行 `acquireQueued()` 的 for 循环，获取前一个节点，判断队头和尝试获锁，成功之后把节点设为队头，并断开原来的队头让其等待 GC

以上为完整的一次线程加锁 -> 线程竞争公平锁 -> 线程排队等锁 -> 线程释放锁 -> 线程排队后等到锁的过程

### 模拟流程
1. T1 执行 lock 获取了锁 AQS.state = 1
2. T2 并发执行，执行到 lock，获取锁失败，初始化队列，为 T1 创造节点，为自己创造节点并加入队尾，再进行一次 CAS 获锁，未成功，将代表 T1 的节点是 waitStatus 设为 SIGNAL(-1) ，T2 park
3. T3 并发执行，执行到 lock，获取锁失败，创造 T3 的节点，加入队尾，判断自己前一个节点是否为队头（即自己是否有资格再尝试 CAS一次），无资格，将代表 T2 的节点是 waitStatus 设为 SIGNAL(-1) ，T3 park
4. T1 任务结束执行 unlock，释放锁，AQS.state=0，unpark T2，T2 进行 CAS 获取锁，AQS.state = 1，T2 把代表它的节点设为队头（空节点）
5. T2 unlock，锁是否，AQS.state=0，unpark T3，T3 CAS 获取锁，AQS.state =1，T3 把其节点设为队头
 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903163323363.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)
## 关于线程 parkAndCheckInterrupt() 的返回值
在线程被 unpark 之后，会 return Thread.interrupted()，并且这个函数会清楚中信号量为 false

```
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```
有经验的人一看就知道这里肯定和线程中断信号设置有关，那假设线程在 lock 阻塞时调用了interrupt 方法进行中断信号的设置，那么这里就会返回 true，同时中断信号 clear 成 false，再向上走：

```
acquireQueued return true -> selfInterrupt() -> Thread.currentThread().interrupt()
```
最后修正了这个线程的中断信号，然后对线程其实并不其任何中断作用，parkAndCheckInterrupt() 方法如果一开始不调用 Thread.interrupted()，其实后续也根本不用调用 Thread.currentThread().interrupt() 去修复，所以是为什么？

原因是为了 `lock.lockInterruptibly()`

```
ReentrantLock.lockInterruptibly -> FairLock.acquireInterruptibly(1)

public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException(); 
        // 该线程已经别设置了中断信号，直接抛出异常等待捕获
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE); // 创建节点初始化队列并加入队列
    try {
        for (;;) {
            final Node p = node.predecessor(); //获取前一个节点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```
与 lock 的逻辑基本相同，但是被 unpark 之后如果 `parkAndCheckInterrupt()` 里面判断为线程被中断，则会抛出“中断异常”，因此 `parkAndCheckInterrupt()` 的写法其实为可中断加锁设计的

## 尝试加锁
```
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
属于Sync 的 final 方法，不可继承和重写，只是实现加锁和重入锁，加锁失败了，直接返回 true，不涉及到队列操作

## 非公平锁
```
 final void lock() {
     if (compareAndSetState(0, 1))
         setExclusiveOwnerThread(Thread.currentThread());
     else
         acquire(1);
 }
```
区别 1：公平锁 lock 之后就直接执行了 `acquire(1)`，在这个方法里面就会先进行队列判断，FIFO 的方式去获得锁；而非公平锁 lock 之后马上抢了一次锁，抢到了就占有，未获得锁再 `acquire(1)`

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
区别 2：`tryAcquire(arg)` 的实现不同， 公平锁的实现如上面所说，优先考虑是否要排队；非公平锁的实现是调用 `nonfairTryAcquire(acquires)`，再一次判断锁状态，想要尝试获取锁，再失败了才会去排队

总的来说

**公平锁**，判断锁状态，尝试获取，失败就排队，排队后发现如果是 2 号位，再尝试一次，失败就 park

**非公平锁**，抢锁，失败就再尝试获取一次，再失败就去排队，排队候发现如果是 2 号位，再尝试一次，失败就 park