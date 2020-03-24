---
layout: post_no_cmt
title:  "Android：Lrucache 原理与小记"
date:   2016-05-21 06:00:00 +0800
categories: [android, java]
---
# 算法
Lru：Last recently used 最近最少使用，即在固定的大小限制下，缓存的 put 和 get 过程中，当 size 超过 maxSize，优先删除那些最久没有访问过的或访问量小的缓存。

# 原理
核心点是使用了 LinkedHashmap：

```
private final LinkedHashMap<K, V> map;
this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
```
参数分别是初始容量，扩展因子，排序模式：true：访问排序；false：插入排序

Lrucache 就是利用 LinkedHashmap 的 “访问排序” 实现了 Lru 算法：

即当调用 put/get 的时候，底层调用了 map 的 put/get，而 put/get 到的数据会移动 map 的第一位，而对于没有访问到数据就会慢慢的沉到末尾，因此当因空间不足移除老数据的时候，trimToSize(maxSize) 方法实际上都是删除 LinkedHashmap 最后一个 item。

## 关于 LinkedHashmap
LinkedHashmap 是 Hashmap 的子类，Hashmap 实现是哈希表结构，节点数组 + 节点链表，是无序的。而 LinkedHashmap 在集成 Hashmap 的基础上，也继承扩展了 Hashmap 的节点 Node 类，使节点类支持了双向链表的数据结构：

```
// hashmap node
Node(int hash, K key, V value, Node<K,V> next) {
    this.hash = hash;
    this.key = key;
    this.value = value;
    this.next = next;
}

// linkedhashmap node
static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    LinkedHashMapEntry<K,V> before, after;
    LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

/**
 * The head (eldest) of the doubly linked list.
 */
transient LinkedHashMapEntry<K,V> head;

/**
 * The tail (youngest) of the doubly linked list.
 */
transient LinkedHashMapEntry<K,V> tail;
```
在使用双向链表而有了有序的功能后，又补充了 get/put 之后节点的链表排序操作，把最近分配的和最近访问的节点都插入到了链表的末尾，老的数据就会慢慢的向头部移动。因此，在 Lrucache 删除末尾老数据的时候，实际就是在删除 linkedhashmap 内部双向链表头部的数据

## LinkedHashmap 补充
- 继承 Hashmap，内部实现依旧是节点数组 + 节点链表 的散列表结构
- 静态内部类 Entry 继承 Hashmap.Node，实现节点的双向链表结构
- 后期又在节点数组后加入了红黑树结构

# 使用
- put/get 缓存的写和读
- 缓存的总大小控制

自主控制缓存总的大小限制，例如 Bitmap 内存缓存的大小控制，只允许保存 4MB 大小的 Bitmap 缓存：

```
int cacheSize = 4 * 1024 * 1024; // 4MiB
LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
   protected int sizeOf(String key, Bitmap value) {
       return value.getByteCount();
   }
}
```
- 下面几种情况下需要主动去释放缓存或者做其他处理的需要重载 `entryRemoved(boolean, K, V, V).`：（evicted 参数是否由于空间不足而需移除老数据）

	- 缓存的在 put 的时候发生替换的时候 (evicted = false)
	- 在缓存总大小超出限制而需要移除老缓存的时候 (evicted = true)，这又细分 4 个情况：
		- put 时候超出限制
		- get 时候发生 cache miss，然后用 create(K) 创建值插入后超出限制
		- resize(maxSize) 重新设置最大容量时候
		- evictAll 移除所有缓存的时候，实际是调用 trimToSize(-1)，把 maxSize 临时改成 -1 然后把所有缓存都已超出最大容量方式移除
	- 在主动移除缓存的时候 (evicted = false)
	- 多线程操作 cache 的时候，利用 create(K) 创造了一个值，但是发现其他线程已经对这个 key 缓存了数据，缓存冲突，需要移除 create(K) 创造了一个值 (evicted = false)

- 当访问缓存，cache miss 的时候，需要统计 miss，甚至直接存入并返回一个值，重载`create(K)`








