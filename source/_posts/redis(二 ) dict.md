---
title: redis(二) dict
date: 2024-10-15 23:07:00
categories: 
- redis
---

本文主要分析redis存储的底层哈希表。即`src/dict.c` , `src/dict.h` 

# 哈希的存储结构
这是最底层的存储kv的结构体dictEntry，中间的`union`可以共享内存，也就是如果是`uint64_t`,  `int64_t`,  `double` 则是直接与key存在一起，用val存储地址。
```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

接着就是一个二维数组, `sizemask`就是所用的取模数，用来求dictEntry的存储位置。
```c
typedef struct dictht {
    dictEntry **table; // 二维表
    unsigned long size;
    unsigned long sizemask; 
    unsigned long used; 
} dictht;
```

最后世界的就存储在`dict`内。
`ht[0]`寸的是实际的数据，`ht[1]`是用来rehash的，rehashidx rehash的时候的桶位置。
```go
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2]; 
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

接下来我们通过解决一些问题来看看源码。
# redis如何解决的哈希冲突？
我们知道常见的哈希冲突就两种解决方案，一种是开放寻址，另外一种就是拉链法。redis选择的是后者。数据结构前面的`dictEntry->next` 就是用来拉链指向下一个桶位置相同的数据。我们可以来看看插入数据是如何处理的。
```c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry.
     * Insert the element in top, with the assumption that in a database
     * system it is more likely that recently added entries are accessed
     * more frequently. */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}
```

我们来看看他的执行步骤。
1. 首先是看key在当前表中是否已经存在、不存在才继续执行
2. 接着获取当前key存储的桶位置
3. 获取成功通过头插法插入数据
4. 最后再赋值`key`给`entry->key`（这里不一定直接赋值），这里后面再看看。
所以最后总结，其实就是一个头插法。

# 拉链长了怎么办？
拉链长了的解决方案很简单，换个大哈希表就行了，也就是`rehash`，这也就是前面的`dict->ht`为什么要两个的原因。
接下来我们主要讨论三个问题？
- 什么时候触发 rehash？
- rehash 扩容扩多大？
- rehash 如何执行？
## 什么时候触发 rehash？
触发rehash的逻辑主要在`_dictExpandIfNeeded`,
```go
/* Expand the hash table if needed */
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}

```
我们可以来看看这个函数的执行逻辑
1. 首先判断目前是否正在rehash，不在rehash才继续进行下面的步骤。
2. 其次就是判断是否是新建dict，即ht[0].size = 0, 这个时候初始化即可。
3. 最后就是判断是否可以rehash了。条件即`d->ht[0].used >= d->ht[0].size`的同时允许resize或者`d->ht[0].used/d->ht[0].size > dict_force_resize_ratio` 这个dict_force_resize_ratio=5，这个时候哈希表负载已经很高了，所以无论是否允许resize都直接rehash了。
第三点也就是触发rehash的前提。其主要就是看哈希表的负载。
## rehash 扩容扩多大？
拓容多大，在前面也提到了，走的`dictExpand`函数，我们可以来看看这个函数。
```c
int dictExpand(dict *d, unsigned long size)
{
    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}

```
1. 首先是判断不能拓容的状态，正处于哈希中或者used > size是不进行拓容的
2. 接着就是就是newhash表的realsize，这个求的逻辑很简单，点进去看看！
3. 如果realsize还是等于原size，则分配失败，return err
4. 接着是创建一个大的哈希表放在d.ht[1]位置上

接下来就是`ht[0]` 数据迁移到 `ht[1]`是怎么操作的？
##  rehash 如何执行？
我们知道redis关于主内存库的操作都是主线程在执行，所以一次性将哈希表拷贝过去肯定是不可能的。所以就产生了`渐进式哈希`的想法。也就是一次迁移一点数据过去。防止阻塞主线程，redis中一切阻塞主线程的行为都是需要减少的，无论是空间上还是时间上。

其主要逻辑在`dictRehash`中，
```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
1. 如果当前没在rehash直接return
2. 接着进入正式流程，若果当前桶是空的empty_visits--同时rehashidx++，empty_visits也是为了避免一直是空桶，阻塞主线程。
3. 接着就开始对这个桶进行迁移到ht[1]上。
4. 这n次迁移操作完成跳出循环后，需要判断迁移是否完成，需要修改rehashidx = -1，这是判断当前dict是否在进行hash的标志

而这个函数的调用者是`_dictRehashStep`, `_dictRehashStep`的调用者则是`dictAddRaw，dictGenericDelete，dictFind，dictGetRandomKey`, 也就是每次对哈希表的查询或修改操作执行一个bucket的迁移。

但是现在存在一个小问题，如果一直没有访问，迁移就不执行了吗？目前而言好像确实是这样的，但是这样数据不是存在两份，内存占用不是会变多吗？redis能忍受这样的内存浪费吗？已提(issue)[https://github.com/redis/redis/issues/13605], 到时候看看后续。




