---
layout: post_no_cmt
title:  "ThreadLocal 原理"
date:   2016-05-16 00:06:00 +0800
categories: [java]
---
Last modified: 2020-04-10

# JDK 文档描述

> This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

ThreadLocal 提供了线程本地变量。这些变量不同于线程里面普通的通过 get/set 访问的那些变量，他们是需要单独初始化的。ThreadLocal 实例在类中通常是私有静态域 "private static"，而这些类的目的是想要关联状态到一个线程上 (例如 User ID 或者 事务 ID)。

> For example, the class below generates unique identifiers local to each thread. A thread's id is assigned the first time it invokes ThreadId.get() and remains unchanged on subsequent calls.

举个例子，下面的类为每个线程生成唯一标识符。线程的标识符调用 `ThreadId.get()` 进行第一次分配，并且在后续的调用上保持不变。

```
 public class ThreadId {
     // Atomic integer containing the next thread ID to be assigned
     private static final AtomicInteger nextId = new AtomicInteger(0);

     // Thread local variable containing each thread's ID
     private static final ThreadLocal<Integer> threadId =
         new ThreadLocal<Integer>() {
             @Override protected Integer initialValue() {
                 return nextId.getAndIncrement();
         }
     };

     // Returns the current thread's unique ID, assigning it if necessary
     public static int get() {
         return threadId.get();
     }
 }
```

> Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).

只要线程活着并且 ThreadLocal 实例可访问，那么每个线程就会持有一个隐式引用，引用指向线程本地变量的副本；在线程完成任务后，所有它的线程本地的副本都会服从垃圾回收机制进行回收 (除非这些副本存在其他的引用指向它们)

# ThreadLocal 关键代码 get/set

```
// get()
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

private T setInitialValue() {
    T value = initialValue(); // 方法重载用于初始化
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

// set()
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
重点在于 `ThreadLocalMap` 以及 `getMap(Thread) & createMap(t, value)`方法

```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
本质上，ThreadLocal 就是在对线程上的成员变量在进行操作，因而 ThreadLocal 的 get/set 本质上是改变线程成员变量，这当然每个线程自有的。

```
// 几个类包含关系图
Thread                                                      
 |
 -- ThreadLocals: ThreadLcaolMap  <-- get/set by ThreadLocal
     |
     -- table: Entry[]
         |
         Entry: WeakReference<ThreadLocal<?>>
```

反之，如果数据不用 ThreadLocal 包裹，那么对于线程来说就应该是共享的。

举例：

```
private static class Member {
    private ThreadLocal<String> belongTo = new InheritableThreadLocal<>();
    private String shared;

    public void setShared(String shared) {
        this.shared = shared;
    }

    public void setBelongTo(String belongTo) {
        this.belongTo.set(belongTo);
    }

    public String getShared() {
        return shared;
    }

    public String getBelongTo() {
        return belongTo.get();
    }
}

// main():
Member member = new Member();
new Thread(new Runnable() {
    @Override
    public void run() {
        member.setBelongTo(Thread.currentThread().getName());
        member.setShared("This is a shared stuff");
        System.out.println(member.getBelongTo() + "; " + member.getShared());
    }
}, "T1").start();
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println(member.getBelongTo() + "; " + member.getShared());
    }
}, "T2").start();

// output:
T1; This is a shared stuff
null; This is a shared stuff
```
结果很显然：belongTo 使用 ThreadLocal 包裹，当它在 T1 设入值，该值只属于 T1，T2 无法读取 (本质上可以说没有)，但是 shared 数据就是共享的，在多线程中涉及到线程安全

# ThreadLocalMap 数据结构

核心部分

```
private Entry[] table;

static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

// set() 
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

// get()
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```
注意点：ThreadLocalMap 并非数组+链表的散列表结构，而是数组结构。
- 计算 index 的 hash 算法 `key.threadLocalHashCode & (len-1)` 这是一种类似取模的操作
- hash 冲突处理，散列表的处理是以当前节点为链表头，向后挂链表节点，而在这，`nextIndex(i, len)` 是单纯的 i + 1(也叫线性探测)，下一个数组元素，再判断非空再进行比较

其他细节：
- set：从 hash 出的下标 i 开始查找
    - 找到相同 key 值的替换
    - 找到 key 值等于 null，但是数组元素非 null 的，说明该元素对应的线程已经完成，key 指向的 ThreadLocal 也已被回收，`replaceStaleEntry(key, value, i)` 除了替换该位置的 key-value 外，还会额外的做其他 "腐坏元素的清理"
    - 由于 tab[i] == null 跳出了循环，插入新的 Entry，更新 size
    - 最后根据如果已经无法再清理 "腐坏元素"，且 size 已经到达临界值 数组长度的 2/3 这两个复合条件，再确定是否进行 resize 和 rehash
- get：相对 set 简单一点
    - 也是从 hash 出的下标 i 开始判断，相同 key 的返回
    - 不相同的 key 的使用线性探测继续查找
    - 找到相同 key 的返回，途中找到 key 为 null 的也进行 "腐坏元素清理"
- 临界值，一开始设置的临界值是数组长度的 2/3，但在触发 rehash() 的时候，再清理完 "腐坏的元素" 后，resize 的触发条件在 `threshold` 的基础上又乘上了 3/4，因此触发临界值应该是 "数组长度的 1/2"

```
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}

private void rehash() {
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```