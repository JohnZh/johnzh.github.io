---
layout: post_no_cmt
title:  "Hashmap ConcurrentHashmap Java7/8 分析与细节记录"
date:   2020-09-10 00:06:00 +0800
categories: [java]

---

# 分析的维度

因为源代码太多，所以从以下几个角度去分析，比较异同，记录细节，且一般情况下 HashMap 只会和 HashMap 比较，ConcurrentHashMap 也是

- 数据结构
- 初始化
- 插入 PUT
  - hash 碰撞
  - 二次 hash、扰动算法
  - 链表转树、树化条件
- 扩容
  - 扩容条件
  - 数据转移
  - 拆树，树转链表、重新树化
- 获取 GET
- 删除 REMOVE
  - 树转链表
- 线程安全的实现
- 线程安全下的 size 计算

# 简称

为了方便，后面 HashMap 简称 hm，ConcurrentHashMap 简称 chm

# hm7 vs hm8

## 数据结构

- 7 数组+链表，8 还多了红黑树，从链表转为树的时候树结构里面还维护了一个双向链表

## 初始化

如果不使用构造器设置，hm7 会提前创建默认值大小的数组，hm8 数组是延时初始化，默认长度也是 1<<4（16），最大值为 1<<30（2^30），加载因子都是 0.75

ps：传入的初始化容量也会向上取到 2 的幂次，eg. 17 实际初始化为 32(2^5)

如果使用参数的构造器初始化，hm7 会先根据参数创建数组；hm8 不会，8 会在 put 的时候再创建

tips：初始化 hashmap 的时候如果知道数据大概的量可以提前设置，这样可以避免扩容转移数据带来的性能损耗

## 插入 PUT

- 插入逻辑基本一致：
  - 数组未初始化，优先初始化
  - key.hash 二次 hash，key 为 null 的插入 0 号位，HM8 的话 hash 就是 0，这意味着后续计算出来的数组下标也是 0 号位
  - 二次 hash 后的 hash 值与（数组长度 - 1） 做二进制 &，计算数组下标
  - 数组节点 null，new 新节点，插入
  - 数组节点不为 null，**即 hash 碰撞**，分两种情况，链表，HM8 里面还有红黑树
  - 先说查找链表，查到了就替换，并返回老值；否则插入，插入之后要做是否树化的判断
  - 树也是先做树查找，找到了会做值的替换，并返回老值；否则做红黑树的插入
  - 最后 modCount++，size++，然后 size 还要和阈值比较，判断是否要扩容

### 二次 hash 扰动算法

扰动算法的作用是对 key 的 hashcode 进行二次 hash，目的是为了让后面与数组长度做 & 运算时候更加的散列。那它是怎么做到这个事情的，大致原理是什么？

答：最初的时候，数组长度很短，比如 16，做 & 运算的时候还要减一，即 15，二进制 1111。新 hashcode 与 1111 做 & 运算的时候参与计算的只有新 hashcode 的后 4bit，4bit 之后的高位就没用了，如果将高位也加入运算，那么这肯定可以减少 hash 碰撞的概率。**扰动算法的核心就是用右移和异或运算，让高位 bit 参与到新 hashcode 的生成**

```
假设两个数据 key.hashcode 为 ：1000 1010，0000 1010
直接与 & 1111 结果都是 1010，hash 碰撞
做一次扰动：h>>>4 ^ h：
1000 1010>>>4 = 0000 1000，然后 0000 1000^1000 1010 = 1000 0010
0000 1010>>>4 = 0000 0000，然后 0000 0000^000 1010 = 0000 1010
新的 hashcode 和 1111 做 &，未发生 hash 碰撞
```

### 插入细节

- hm7 插入节点是用头插，hm8 插入节点是用尾插。头插较尾插效率上有一定优势
- hm7 的二次 hash 算法更复杂，散列性更强（hash 碰撞概率更小），hm8 的二次 hash 算法非常简单（hash 碰撞概率较 hm7 高）
- hm8 加入了红黑树，即使 hash 碰撞概率高了，尾插效率低了，但是一旦链表过长就会树化，那么查询效率还是提高了

```
// 头插，每次都是第一个节点，之前的第一个节点成为构造器的 next 入参
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    if (size++ >= threshold)
        resize(2 * table.length);
}
```

### 树化条件

代码中树化阈值常量为 8，但是执行中其实为链表节点为 9 的时候才会进行树化，并且在树化之前还会先判断数组长度，大于等于 64 才会去树化，小于的情况会先去扩容数组

```
// TREEIFY_THRESHOLD 为 8
for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) { // p 为数组节点
        p.next = newNode(hash, key, value, null);
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        break;
    }
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
        break;
    p = e;
}
```

### 链表转树

从数组节点（头节点）开始，遍历一次，把节点都转成 TreeNode，同时生成双向链表，然后再做第二轮遍历，生成红黑树。

红黑树生成细节：红黑树是特殊平衡二叉树，因此也满足另一节点，其左子树均小于该节点，其右子树均大于该节点，因此在插入过程中有一个判断大小的条件：

- 先判断 hash 值
- key 如果实现了 Comparable，使用这个接口提供的方法判断
- 未实现接口，或者实现 Comparable 的 key 比较之后相等，然后再用 key 的类名长度比较
- 依然相等就用 key 类名的系统 hash 值，等于也当做小于处理

生成树后，将树的 root 节点返回赋值给数组节点（因为树在生成的过程中会进行平衡算法，其 root 节点不一定就是之前的第一个节点）

```
moveRootToFront(tab, root);
```

#### 红黑树插入平衡算法

红黑树的 5 大性质：

- root 节点为黑
- 节点只有红和黑
- 红节点的子节点是黑色
- null 为黑
- 黑高相同，即从任一个节点遍历到叶子节点，黑色节点个数相同

插入和平衡调整：（p 为父节点，pp 为爷爷节点，u 为叔叔节点，x 为当前操作节点）

- 插入节点都为红
- 插入节点的 p 为黑，无变化
- 插入节点的 p 和 u 为红，p，u，pp 都反色，再以 pp 为 x 再调整
- 无 u，p 为 pp 的左子节点，且为红
  - 插入为 p 的左节点，pp 为 x 进行右旋
  - 插入为 p 的右节点，以 p 为 x 进行左旋，再以 pp 为 x 右旋
- 无 u，p 为 pp 的右子节点，且为红
  - 插入为 p 的右节点，pp 为 x 进行左旋
  - 插入为 p 的左节点，以 p 为 x 进行右旋，再以 pp 为 x 左旋

## 扩容

hm7、8 都是扩容为原大小的两倍

### 扩容条件

hm7 的扩容条件需要满足大于阈值和当前插入节点发生 hash 碰撞才扩容
hm8 的扩容条件只有大于阈值这一项

（扩容上限 1<<30 就不提了）



### 数据转移

流程是一致的，先扩容，再转移：

- hm7：遍历数组，对数组节点用扩容后的长度重新做一个 &，得到新的数组下标，转移，然后取数组节点的 next，如果不为 null 即为链表，继续 & 运算计算新下标，做头插。最后赋值新数组，计算新阈值。
- hm8：对于单纯数组节点，和 7 一样，重新计算下赋值即可。链表，转移之前会做一个拆表的过程，把整个链表拆为低位表和高位表，一次性把低位和高位表移到新数组对应的下标去。低位表的下标就是当前老数组节点的下标，比如说 j，高位表的就是 j + oldCap（老数组的长度）

```
// hm8 拆表
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
Node<K,V> next;
do {
    next = e.next;
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
    else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```

拆表的关键点在于 e.hash & oldCap 的值是否等于 0，解释：

```
假设老数组的长度为 16，扩容后为 32，那么做 & 运算的时候使用的就是 31，二进制 1 1111

重新 hash 之后如果说下标会发生变化的元素，其 hash 值必定满足 1 1111<=hash<=1 0000 
因此和 16（1 0000）做 & 必然不等于 0

而下标不会变的元素的 hash 和 16（1 0000）做 & 必然等于 0
```

### 拆树，树转链表，重新树化

那对于 hm8 来说，除了链表还有树，树在扩容转移的过程中也会发生拆树，树转表的情况：

首先，从 root 节点开始遍历，更加高低位拆表算法，把树内部的双向链表拆成低位和高位
链表。

其次，低位或高位链表如果节点小于等于 6 个，那么这个时候要**树转链表**，操作这两个链表分别转化节点为链表节点，然后分别挂到 j 和 j+oldCap 的新数组处。

如果，低位或高位链表节点大于 6，需要判断另一个链表是否存在，不存在，说明当前这条链就是一颗完整的树，直接将 root 赋值到新数组对应位置上；存在的情况，**重新树化**这条双向链表。

## 获取 GET

hm 的 GET 相对还是简单的，计算二次 hash，然后是数组的遍历，链表的遍历，树的遍历，找到返回即可

## 删除 REMOVE

hm7、8 数组的删除和链表的删除都比较简单。hm8 中还需要做红黑树的删除，删除完还要进行树的平衡调整，同时也要维护好双向链表，当删除到某个条件下还要进行树转链表。

（红黑树的节点删除比较复杂，先不展开，大致就是先找到直接前继或后继叶子节点，然后交换数据，然后删除叶子节点，当删除黑色节点的情况还要进行平衡调整，平衡调整又分为很多情况）

### 树转链表的条件

```
if (root == null || root.right == null ||
    (rl = root.left) == null || rl.left == null) {
    tab[index] = first.untreeify(map);  // too small
    return;
}
```

满足这个条件的最小情况，root 节点的左子节点的左节点为空，满足这个条件的树实际上可以是以下几种节点数量的情况：

- 还剩 6/5/4个节点时候**可能**会发生转链表
- 小于等于 3 个节点必定转链表

## 线程安全和 size 计算

hm 是非线程安全的，因此不讨论，只是提及一下，hm 在多线程中会发生循环链表的情况。

size 也是在每次操作后记录的，因此获取一下这个成员变量就好了

# chm7 VS chm8

## 数据结构

chm7 使用的 Segment[] + HashEntry[] 这样的数据结构，即维护一个 Segment 数组，每个 Segment 里面维护一个 HashEntry 数组。仿佛是每个 Segment 里面一个小 hashmap

- Segment 继承 ReentrantLock，在进行类似于 PUT/REMOVE 这种一连串的大操作的时候会先加锁，再操作。
- 而其他不是特别复杂的操作，比如 GET 或 SIZE 会使用 cas 或者 Unsafe 提供的其他一些方法来完成

chm8 则不再使用 7 里面的数据结构，还是直接使用 Node[]，复杂的操作使用 synchronized 关键字加锁完成，非复杂的操作也是 cas 获取 Unsafe 提供的其他方法。

- 树的操作，提供了 TreeBin 和 TreeNode，TreeBin 对象里面维护着 root 的 TreeNode。因为在树的平衡算法中 root 会发生改变，在 root 外再套个对象就避免了数组和 root 节点的赋值操作（数组节点一直指向 TreeBin 对象）

## 初始化

chm7 由于多了 Segment 这一层，因此在初始化的时候除了初始容量，加载因子，还多了一个**同步等级：concurrencyLevel**，这个参数和 Segment[] 长度有关

**ps：**chm8 里面虽然也有 concurrencyLevel，但是实际上已经没有 7 里面的那个作用了，它在 8 里面的作用就是：

```
if (initialCapacity < concurrencyLevel)   // Use at least as many bins
    initialCapacity = concurrencyLevel;   // as estimated threads
```

初始容量和同步等级都会向上取整为 2 的幂次

```
int sshift = 0;
int ssize = 1;
while (ssize < concurrencyLevel) {
    ++sshift;
    ssize <<= 1;
}
this.segmentShift = 32 - sshift;
this.segmentMask = ssize - 1;
if (initialCapacity > MAXIMUM_CAPACITY)
    initialCapacity = MAXIMUM_CAPACITY;
int c = initialCapacity / ssize;
if (c * ssize < initialCapacity)
    ++c;
int cap = MIN_SEGMENT_TABLE_CAPACITY; // MIN_SEGMENT_TABLE_CAPACITY 为 2
while (cap < c)
    cap <<= 1;
    
// create segments and segments[0]
Segment<K,V> s0 =
    new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                     (HashEntry<K,V>[])new HashEntry[cap]);
Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
this.segments = ss;
```

代码说明：ssize 就是同步等级向上取的 2 的幂，cap 代表一个 Segment 里面 HashEntry[] 的长度，最小值是 2，最初会由 initialCapacity / ssize 得来，但是经过**“向上取整”“和最小值比较并向上取 2 的幂”**的操作后才是最终数据：

```
比如 initialCapacity：33，concurrencyLevel：15
concurrencyLevel：15 while 循环取 2 的幂得 16
cap 的初值 c = initialCapacity / ssize  = 33 / 16 = 2 
向上取整： if (c * ssize < initialCapacity) ++c; 于是 c：3
和最小值比较后并向上取 2 的幂：3 比 2 大，但是还要取 2 的幂，cap 最后等于 4

最后结果：Segment[] 长度 16，Segment 中 HashEntry[] 长度 4
```

s0 的作用，s0 放在 Segment[0] 作为后续插入数据时候用于生成对象的模板（PUT 中生成 Segment 时候会用到）

相比于 chm7 这么复杂的初始化过程，**由于 chm8 数据结构更像是 hm8，因此初始化也非常简单，流程类似于 hm8，都是在 put 的时候才真的去初始化数组，有参构造器只是对几个成员变量进行初始化**



## chm7 chm8 操作分记

由于数据结构的不同，实现的算法也不同，操作上分开记录会比较清楚

## chm7 插入 PUT

插入核心逻辑与 hm 是一致的，除了 chm 不支持 null-key，chm7 采用了**分段加锁**保证线程安全

- 通过 key 的 hashcode 二次 hash（还会进行 hash >>> segmentShift，segmentShift = 32 - sshift，相当于取了高位 bit），然后与 & segmentMask（Segment[] 长度减 1） 得到 Segment[] 下标，再对这个位置的 Segment 做 put

- 当前之前还要确保这个 Segment[] 节点的存在，ensureSegment。这个方法使用了 UNsafe.getObjectVolatile 直接读内存 + CAS的方式来为未分配内存的 Segment[] 节点进行初始化的工作

  ```java
  private Segment<K,V> ensureSegment(int k) {
      final Segment<K,V>[] ss = this.segments;
      long u = (k << SSHIFT) + SBASE; // raw offset
      Segment<K,V> seg;
      if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
          Segment<K,V> proto = ss[0]; // use segment 0 as prototype
          int cap = proto.table.length;
          float lf = proto.loadFactor;
          int threshold = (int)(cap * lf);
          HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
          if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
              == null) { // recheck
              Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
              while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                     == null) {
                  if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                      break;
              }
          }
      }
      return seg;
  }
  ```

  > 补充一下 long u = (k << SSHIFT) + SBASE 以及 UNSAFE.getObjectVolatile(ss, u) 来得到 ss 数组的第 k 个节点的算法：
  >
  > ```java
  > Class sc = Segment[].class;
  > SBASE = UNSAFE.arrayBaseOffset(sc); // 获取第一个元素在内存中的偏移量
  > ss = UNSAFE.arrayIndexScale(sc); // 数组单个元素大小
  > SSHIFT = 31 - Integer.numberOfLeadingZeros(ss);	
  > // numberOfLeadingZeros 返回的是 ss 二进制数据里面前导 0 的个数
  > // 比如 4（100），前导 0 个数就是 32 - 3 = 29
  > // SSHIFT 计算本来应该是 32 - 1 - Integer.numberOfLeadingZeros(ss) 
  > // 目的是计算出 ss 是 2 的 SSHIFT 次幂，即 1 << SSHIFT
  > long u = (k << SSHIFT) + SBASE;
  > // 起始偏移量（SBASE） + k * (单个元素大小1 << SSHIFT) = 第 k 个元素的起始偏移量
  > ```
  >
  > 

- 进行 Segment.put 的时候首先 tryLock() 尝试加锁，失败则进入**循环尝试加锁**（scanAndLockForPut）。scanAndLockForPut 做了这几件事：

  - 通过 hash 找到 HashEntry[] 节点 first，null，创建一个，后续得到锁后可以直接插入
  - first 有了，并且 key 和我现在要插入的相同，改值的情况，继续循环抢锁
  - first 有了，key 和我不一样，next 节点，循环继续上面两个判断
  - 创建了节点，或者找到了节点之后循环重试次数大于 MAX_SCAN_RETRIES（多核 64，单核 1），循环扫描够了，调用 lock，再抢一次锁，抢到不就阻塞
  - 循环扫描次数没达到最大，重试次数偶数次，并且 HashEntry[] 节点 first 发生了变化（这个情况是由于其他线程 remove 或者 put 了新节点进来），重新赋值 first，回到最开始的状态重来
  - 抢到锁了，返回创建的新节点 node，或者 null（修改节点的情况）

- 接着就在 lock-unlock 代码块内做类似 hm7 put 逻辑的原子操作：

  -  HashEntry[] 节点 null，头插
  -  HashEntry[] 节点不为 null，向 next 遍历直到 nex 为 null，找相同 key 的替换，也可以不替换（**chm7 多了 putIfAbsent 方法，onlyIfAbsent 变量会为 true，不会替换老的值**）
  -  到最后都找不到相同的，头插
  -  计数器加 1，判断是否大于阈值，大于则扩容

## chm7 扩容

Segment[] 的长度是始终不变的，扩容是 double 当前 Segment 的 HashEntry[] 的长度。扩容的上限和 hm 一样：1<<30

### 扩容条件

条件就是大于阈值

### 数据转移

- 老的 HashEntry[] 就单个节点的，直接重新计算下标赋值到新数组

- HashEntry[] 节点是链表的，它做了一个优化操作，从头到尾遍历，先找到一段（运气不好就一个）会移动到相同下标的链，然后移动这段链到新数组。接着，再从头到这段链的第一个节点进行遍历，一边重新 hash 一边头插节点到新数组

  ```java
  HashEntry<K,V> lastRun = e;
  int lastIdx = idx;
  for (HashEntry<K,V> last = next;
       last != null;
       last = last.next) {
      int k = last.hash & sizeMask;
      if (k != lastIdx) {
          lastIdx = k;
          lastRun = last;
      }
  }
  newTable[lastIdx] = lastRun;
  // Clone remaining nodes
  for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
      V v = p.value;
      int h = p.hash;
      int k = h & sizeMask;
      HashEntry<K,V> n = newTable[k];
      newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
  }
  ```

  

## chm7 获取 GET

并未加锁，而是直接使用 Unsafe.getObjectVolatile 获取到主存对应的 Segment，在用  Unsafe.getObjectVolatile 获取到 HashEntry[] 对应数组节点，然后直接遍历链表获取值

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key.hashCode());
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

这段代码其实只保证了读的内存可见性，那么其他线程在写的时候也必须保证其写的可见性，不然还是会出脏数据



## chm7 删除 REMOVE

类似于 PUT，先加锁，加锁失败循环扫描再加锁 scanAndLock。加上锁了，就在 lock-unlock 代码块之间做原子删操作。



## chm7 线程安全的实现

总的来说 chm7 的线程安全使用的技术是 ReentrantLock，毕竟 Segment 继承 ReentrantLock，然后配合 UNsafe 直接对内存的操作，保证了多线程操作变量的可见性，以及在初始化 Segment 的时候使用了自旋来保证了线程安全



## chm7 线程安全下的 size 计算

循环遍历每个 Segment，sum 值累加每个 Segment 的 modCount（操作次数），期间也会统计 Segment 里面 put 进去元素个数（size += seg.count），循环结束，判断 sum 和 last 是否相等（last 一开始的 0），last 赋值 sum

进入第二次循环，一样的过程，不过这次 last 如果没有其他线程干扰就可能和 sum 相等，相等就跳出循环，返回 size

不相等，进入第三次循环，一样的过程，还不相等，第四次循环同步（lock），lock 之后再进行 size 的统计

## chm8 插入 PUT

- 不允许 null 的 key 和 value

- 二次 hash，算法：右移 + 异或，再 & HASH_BITS（除 32bit 是 0，其他 bit 都为 1），保证正数

  ```
  (h ^ (h >>> 16)) & HASH_BITS
  HASH_BITS = 0x7fffffff
  ```

- 接着是自旋（死循环+CAS）

  - 如果数组 null，初始化数组，initTable，也是自旋，通过 **sizeCtl** 实现类似锁的机制
    - sizeCtl 初始化是 0，线程进入初始化代码会先把它的值改为 -1
    - 别的线程来初始化的时候，发现sizeCtl 是-1，会进行 Thread.yield，放弃 cpu
    - 初始化完成，sizeCtl 修改为阈值：n - (n >>> 2)，即 0.75*n
  - 数组不为 null，但是数组节点（或叫桶节点），CAS 插入新节点，插入成功跳出循环
  - 当前桶节点的 hash 为 MOVED，说明当前在扩容后的数据迁移，帮助迁移 helpTransfer
  - 其他情况 synchronized 同步后进行类似于 hm8 的原子操作，同样的也有树化条件和链表转树的过程

### 树化条件和链表转树

树化条件和 hm8 一样是链表节点大于等于 8，但是实际上是依然是链表总节点为 9 个才会去转树，即它的这个 8 是除了桶节点之外，链表上的节点数大于等于 8 就转树。同样的在转树之前也要判断过如果数组长度小于 64，那么依然不会转树，先进行扩容



转树的逻辑基本上与 hm8 相同，唯一的区别就是添加了 treeBin结构，也是继承 Node 的，某个下标的数组引用（指针）会指向这个 treeBin，treeBin 包含红黑树的 root，由于在树的插入和删除过程中由于平衡算法，root 会发生变化，这样的结构可以不用每次调整完后都把 root 赋值给数组引用（指针）



### 数据个数的统计

相关方法 addCount、fullAddCount

使用了 baseCount + CounterCell[] 的组合来进行多线程的个数记录。大致的原理：

- 首先会对 baseCount 进行 CAS  + 1，如果失败，会进入 fullAddCount ，初始化一个 CounterCell[] 数组，最开始是大小是 2，使用 `ThreadLocalRandom.getProbe()` 对当前线程生成一个随机数（**重复调用该方法随机数生成相同**），用随机和 CounterCell[].len - 1 做 & 得到下标，创建新 CounterCell 对象并初始化成 1。

- **注意：在 fullAddCount 里面的 CounterCell 数组的创建和扩容，CounterCell 节点的创建，都会使用一个 cellsBusy volatile 变量进行类加锁操作，即只有 CAS(cellsBusy, 0, 1) 成功才能继续执行；但是对 CounterCell 节点内容的CAS 更改不需要 CAS(cellsBusy, 0, 1)**

  >原因：因为对象的创建并非原子操作，细分为，类加载检查，分配内存，赋零值，设置对象头，调用 init，关联引用。
  >
  >```java
  >public static void main(String[] args) {
  >	Object o = new Object();
  >}
  >// 字节码：
  >0 new #2 <java/lang/Object>
  >3 dup
  >4 invokespecial #1 <java/lang/Object.<init>>
  >7 astore_1
  >8 return
  >```
  >
  >调用 init 之前的步骤都是 new 那个指令，期间会有一个赋零值，这个时候如果被别的线程访问并发生修改操作，那在调用 init 的时候，数据就错了！

  **因此切记，使用 new 创建对象的时候一定得加锁或类似加锁**

  

- 其他线程来进行个数统计的时候，如果 CounterCell[] 已经有，用随机数做 &之后得到下标，如果节点不存在，进入 fullAddCount ；如果存在，就会直接做 CAS + 1，失败又会进入 fullAddCount。

- 其他线程进入 fullAddCount，节点不存在的，创建并初始化。节点存在的，再次 CAS + 1，成功就 break 了（忘了说 fullAddCount 里面是个大循环）。再次失败，竞争太激烈了，2 倍大小扩容 CounterCell[] 数组（扩容之前会先检查是否已经被别的线程发生了扩容，是的话就进行下一次循环，不会去扩容）

- 当然 CounterCell[] 数组也不是无限扩大的，当数组长度大于等于可用 cpu核心数的时候，循环就再不会进到扩容的逻辑里面，下面代码就是这个逻辑

  ```java
  ...
  else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
    break;
  else if (counterCells != as || n >= NCPU)
    collide = false;            // At max size or stale
  else if (!collide)
    collide = true;
  else if (cellsBusy == 0 &&
           U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
    try {
      if (counterCells == as) {// Expand table unless stale
        CounterCell[] rs = new CounterCell[n << 1];
        for (int i = 0; i < n; ++i)
          rs[i] = as[i];
        counterCells = rs;
      }
    } finally {
      cellsBusy = 0;
    }
    collide = false;
    continue;                   // Retry with expanded table
  }
  ```



###  数据个数的计算

从上面的数据结构上看，计算就很简单了，把 baseCount 以及 CounterCell[] 全部数据加起来就可以了

```java
public int size() {
  long n = sumCount();
  return ((n < 0L) ? 0 :
          (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
          (int)n);
}	

final long sumCount() {
  CounterCell[] as = counterCells; CounterCell a;
  long sum = baseCount;
  if (as != null) {
    for (int i = 0; i < as.length; ++i) {
      if ((a = as[i]) != null)
        sum += a.value;
    }
  }
  return sum;
}
```



### 补充

chm8 的个数统计原理还可以参考 LongAdder 的原理说明



## chm8 扩容

扩容的入口是在 addCount 里面，check 除了 REMOVE 为-1，其余情况都满足大于等于 0，然后自旋设置 sizeCtl 的值为一个负数，进入 transfer 方法

> 补充一下 sizeCtl 的值和其代表的状态：
>
> - 默认是 0，初始化数组的时候会 CAS 成 -1，作为锁标记
> - 初始化数组结束后会赋值为阈值，正数
> - 满足扩容条件，即数据个数大于阈值，会 CAS 成为一个负数：resizeStamp(n) << 16 + 2
>   - resizeStamp(n) 会用 n 的 2 进制数的前导 0 个数，与（|）上 1 << 15，因此总共左以了 31 位
>   - 最高位第 32位为 1，负数
> - 因次，sizeCtl 为非 -1 的负数表示正常数据转移，且这个数先记作 sc
> - 一个新线程来帮助转移数据，就会 CAS(sizeCtl, sc, sc + 1)，+ n 就代表有 n 个线程在帮助转移数据
> - 线程帮助结束，又会 CAS(sizeCtl, sc, sc - 1)
> - 直到 sizeCtl 恢复到最开始的值（resizeStamp(n) << 16 + 2），说明转移全部结束

扩容大小和 hm8 一样，两倍

### 扩容条件

扩容条件和 hm8 一样，大于当前阈值就扩容

### 数据转移

代码逻辑非常复杂，只能先记录原理。

首先进入 transfer 的线程会进行新数组的创建，然后根据步长计算好自己操作转移范围，步长最小为 16，数据的转移会从 n - 1（原数组的长度 n） 开始到自己操作范围的边界去转移数据。比如，当前当前数组长度为 32，那么当前线程的转移访问就是数组下标 32-1 到 32-16，即 31-16。

> 这里对应几个变量的记录，需要结合代码查看，32 就是老的数组长度，一开始 transferIndex 的大小
>
> 步长 stride 为 16，下一次的边界 nextBound = nextIndex - stride，nextIndex 先会赋值 transferIndex，于是得出 nextBound = 16，而这个值会 CAS 给 transferIndex。
>
> 当前线程的边界 bound = nextBound（16），当前线程开始转移的下标 i = nextIndex - 1（16 - 1），同时 advance 标记为 false，跳出 while 循环进入 for 循环执行数据转移
>
> 如果现在又新的线程来了，它的 nextIndex 就会先赋值为 transferIndex（16），nextBound = 0，bound = nextBound，i = nextIndex - 1（15），advance = false，跳出 while 进入 for 执行数据转移

数据是如何转移的呢，每转移完一个桶（不管桶是单个节点还是链表还是树）都会把这个节点赋值为 ForwardingNode。其他线程如果发现这个地方是 ForwardingNode，做操作的线程就会来帮助转移，在帮助转移的节点就会继续前进（advance = true）



### 转移细节、拆表、拆树

转移的数据的过程也是需要 synchronized 加锁的。

数组节点或者链表，和 hm8 一样，需要进行高位链和低位链的拆分，不仅如此，还结合了 chm7 使用的转移算法。找到最后面会移动到相同位置的一段链，然后根据链头的 hash 和老数组的长度做 & 之后的结果判断是高位链还是低位链，然后从数组节点开始遍历到这段链的链头，根据节点 hash & 老数组长度来判断是高位还是地位，并做头插（头插会改变顺序，区别于 hm8 拆表时候的尾插）



拆树。和 hm8 类似，根据红黑树里面的双向链表结构进行遍历，然后拆成高位链和低位链，同时还记录了两链边长度。根据长度是否小于等于 6，满足条件，树转链表（节点类型转换），不满足判断另外一个链是否存在，不存在，当前就是完整的树，移过去就完了。存在，用链表结构重新树化一边



## chm8 获取 GET

GET 逻辑相对简单：

- 二次 hash，计算下标，然后分别是下面的查找
- 找到数组节点
- 红黑树的查找
- 链表的查找



## 删除 REMOVE

删除的逻辑也在一个大循环里面

- 首先二次 hash，进入循环
- 空表，空节点，直接break
- 找到的节点 hash 为 MOVED 说明在转移数据，帮助转移

- synchronized 桶节点加锁，分别进行桶节点的删除，链表节点的删除，树节点的删除
- 最后调用 addCount(-1L, -1) 更新数据个数



树转链表的条件也是和 hm8 一样的



## 线程安全的实现

总的来说，chm8 在 PUT 和 REMOVE 的复杂操作上或者说是代码逻辑粒度比较大的地方，使用了 synchronized 进行线程安全的加锁。而在代码逻辑粒度比较小的地方，基本上使用了自旋的方式实现线程安全