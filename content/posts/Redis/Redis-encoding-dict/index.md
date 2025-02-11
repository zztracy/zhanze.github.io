---
date: "2025-01-13"
draft: false
title: "Redis 数据编码 Dict"
---

# 概述

在 Redis 中，Dict 是一种核心的数据结构，用于实现键值对（key-value）存储。它是 redis 高效运行的基础之一。在数据库的键空间、Hash 数据类型、有序集合等场景均存在应用。

# Dict的数据结构

Dict由三部分组成：

1. 哈希表节点：dictEntry，表示一个键值对；包含key、value 以及指向下一个节点的指针
2. 哈希表：dictht，包含一个 dictEntry 指针数组，用于存储键值对，以及记录哈希表的大小，已用节点数等信息
3. 字典：dict，包含两个哈希表（ht[0] 和 ht[1]），这样的设计能够支持渐进式 rehash

```c
# dict.h
typedef struct dictEntry {
	void *key; // 键
	union {
		void *val;
		uint64_t u64;
		int64_t s64;
		double d;
	} v;
	// 下一个Entry的指针
	struct dictEntry *next;
} dictEntry;
```



```c
# dict.h
typedef struct dictht {
  // 哈希表数组，每个元素都是指向 dictEntry 的指针
	dictEntry **table;
  // 哈希表的大小，即table 数组的长度
	unsigned long size;
  // 哈希表大小掩码，用于计算索引，值为 size-1
	unsigned long sizemark;
  // 已使用的哈希表节点数量
	unsigned long used;
} dictht;
```



```c
// dict.h
typedef struct dict {
	dictType *type; // dict类型，内置不同的hash函数
	void *privadata; // 私有数据，在做特殊hash运行的时候用
	dictht ht[2]; // 一个dict包含两个hash表，其中一个是当前数据，另外一个一般为空，rehash的时候使用
	long rehashidx; // rehash的进度，-1标识未执行
	int16_t pausereshah; // rehash是否暂停，1为暂停，0为继续
} dict;
```

# 2 Dict的扩容
Dict中的HashTable就是数组结合单向链表的实践，当集合中元素较多的时候，必然导致哈希冲突增多，链表过长，则查询效率会大大降低。
Dict在每次新增键值对的时候都会检查负载因子（LoadFactor = used/size），满足以下两种情况时会出发哈希表扩容

* 哈希表的LoadFactor >= 1，并且服务器没有执行BGSAVE或者是BGREWRITEAOF 等后台进程
* 哈希表的LoadFactor > 5;
```c
static int _dictExpandIfNeeded(dict *d) {
	// 如果正在rehash中，返回ok
	if (dictisRehashing(d)) return DICT_OK;
	// 如果哈希表为空，则初始化哈希表
	if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE)
	// 如果哈希表的用量大于哈希表的大小并且 （dict_can_resize || 负载因子 5）进行扩容
	// dict_can_resize表示当前没有进行bgrewrite等消耗cpu的操作
	if (d->ht[0].used >= d->ht[0].size &&
	(dict_can_resize || d->ht[0].used/d->ht[0].size > dict_force_resize_ratio)) {
		// 扩容大小为used + 1，底层会对扩容大小做判断，实际上找的是第一个大小大于等于 used+1 2 2^n
		return dictExpand(d, d->ht[0].used + 1);
	}
	return DICT_OK;
}
```

# 3 Dict的缩容
Dict除了扩容以外，每次删除元素的时候，也会对负载因子做检查，当LoadFactor < 0.1的时候，会做哈希表收缩
Dict删除元素
```c
// t_hash.c # hashTypeDeleted()
if (dictDelete((dict*)o->ptr, field) == C_OK) {
	deleted = 1;
	// 删除成功后，检查是否需要重置Dict的大小，如果需要则调用dictResize方法进行重置
	if (htNeedsResize(o->ptr)) dictResize(o->ptr);
}
```

```c
int dictResize(dict *d) {
	unsigned long minimal;
	if (!dict_can_resize || dictIsRehashing(d))
		return DICT_ERR;
	// 初始化minimal的值为哈希表的使用量
	minimal = d->ht[0].used;
	// 如果minimal比初始化容量还小
	if (minimal <DICT_HT_INITAL_SIZE)
	// 将minimal复制为初始化大小
	minimal = DICT_HT_INITIAL_SIZE;
	// 调用dictExpand方法，底层执行的时候是第一个大于等于minimal的2^n
	return dictExpand(d, minimal);
}
```

```c
// server.c
int htNeedsResize(dict *dict) {
	long long size, used;
	size = dictSlots(dict);
	used = dictSize(dict);
	// size > 4 并且负载因子低于0.1的时候需要进行缩容
	return (size > DICT_HT_INITIAL_SIZE && (used * 100/size < HASHTABLE_MIN_FILL));
}
```
不管是扩容还是收缩，必定会创建新的哈希表，导致哈希表的size和sizemask会发生变化，而key的查询与sizemask无关。因此必须对哈希表中的每一个key重新计算索引，插入新的哈希表，这个过程称为rehash
1. 计算新hash的realSize，值取决于当前要做的是扩容还是收缩
	* 如果是扩容，则size为第一个大于dict.ht[0].used + 1 的2*n
	* 如果是收缩，则新size为第一个大于等于dict.ht[0].used 的 2*n(不得小于4)
2. 按照新的realsize申请内存空间，创建dictht，并且赋值给到dict.ht[1]
3. 设置dict.rehashidx = 0，标志开始rehash
4. 每次执行增删改查操作的时候都对哈希表的一个角标对应的链表的元素进行rehash，然后rehashindex++，直到将所有的数据都rehash完成。这个操作是迁移！！！！不是复制！！！
5. 将dict.ht[1]赋值给到dict.ht[0]，给dict.ht[1]初始化为空哈希表，释放原来dict.ht[0]的内存
# 4 总结
Dict的结构
* 类似Java的HashTable，底层是数组加链表来解决哈希冲突
* Dict包含两个哈希表，ht[0]平常用，ht[1]用来rehash
Dict的伸缩
* 当LoadFactor大于5或者LoadFactor大于1并且没有子进程任务时，Dict扩容
* 当LoadFactor小于0.1的时候，Dict收缩
* 扩容大小为第一个大于等于used + 1的2*n
* 缩容大小为第一个大于等于used的2*n
* Dict采用渐进式rehash，每次访问Dict时执行一次rehash
* rehash时ht[0]只减不增加，新增操作只在ht[1]进行，其他操作在两个哈希表进行