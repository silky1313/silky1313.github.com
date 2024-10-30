---
title: Redis-存储底层数据结构-ziplist
date: 2024-10-28 23:07:00
categories: 
- redis
---

# ziplist.c的开头注释
这一段注释大概解析了一下ziplist基本构成。

```
ZIPLIST OVERALL LAYOUT
======================
ziplist的布局大概如下：
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

<uint32_t zlbytes> 是无符号整型，表示的是ziplist占用的字节数，包括zlbytes本身。这个值需要存储才能才能调整整个结构的大小而无需先遍历他。
 
<uint32_t zltail> 是最后一个节点的偏移量，这允许pop操作而无需遍历整个ziplist。

<uint16_t zllen> 是entryies的数量，当数量大于2^16-2，这个值只会是2^16-1并且我们需要遍历这个ziplist才知道有多少entire。

<uint8_t zlend> 是一个特殊的entry代表着ziplist的结尾。编码为255的单个字节。没有其他正常entry开始是255.

ZIPLIST ENTRIES
===============
每一个在ziplist中的entry都包含两条带信息的元数据作为前缀。首先，包含前一个节点的长度以便于从后向前遍历。其次，提供了entry的编码。他代表这个entry的类型，整型或者string，对于string来说，他还代表字符串负载的长度。因此完整的entry大致如下：

<prevlen> <encoding> <entry-data>

有时编码代表条目本身，比如最小的整型我们稍后将会看到.这个时候<entry-data>部分发生了丢失，我们可以只使用：
<prevlen> <encoding>

前一个节点的的长度，<prevlen>，他的编码如下：如果前entry的长度小于254个字节，他只需要一个字节的8位无符号整型来代表前entry的长度，当前entry的长度大于等于254时，则需要五个字节。
第一个字节设置为 254 (FE)，表示后面跟着一个更大的值。其余 4 个字节将前一个条目的长度作为值。

so实际编码如下
<prevlen from 0 to 253> <encoding> <entry>

或者如果前一个entry的长度大于253个字节他的编码如下：
0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry>

这个encoding字段取决于entry的实际内容. 当这个entry是string时，encoding的第一个字节的前两个bits用来存储string的编码类型，后面是字符串的实际长度。当这个entry是integer时，前两bit都被设为1.接着的两位被用来指定integer的类型。以下是不同编码类型的概述，第一个字节始终足够用来表示什么类型的entry。

|00pppppp| - 1 byte
string 的长度小于63字节（6 bits）,
"pppppp" 代表6位无符号长度。

|01pppppp|qqqqqqqq| - 2 bytes
String 长度小于16383字节，(14 bits).
IMPORTANT: 这14位以大端形式存储。

|10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
string长度大于等于16384字节，仅第一个byte后面的四个byte代表长度最大2^32 - 1.第一个字节的后留个bit未被使用，设置为0.
IMPORTANT: The 32 bit number is stored in big endian.

|11000000| - 3 bytes
Integer encoded as int16_t (2 bytes).

|11010000| - 5 bytes
Integer encoded as int32_t (4 bytes).

|11100000| - 9 bytes
Integer encoded as int64_t (8 bytes).

|11110000| - 4 bytes
Integer encoded as 24 bit signed (3 bytes).

|11111110| - 2 bytes
Integer encoded as 8 bit signed (1 byte).

|1111xxxx| - 这种格式的描述是针对表示立即 4 位整数的情况。在这种情况下，`xxxx` 是一个 4 位二进制数，可以表示从 0000 到 1101 的无符号整数，范围是 0 到 13。但需要注意的是，由于 0000 和 1111 不能被使用，实际上编码值是从 1 到 13，而不是从 0 到 12。也就是说，编码的值需要减去 1 才能得到正确的值。

|11111111| - End of ziplist special entry.

一个实际ziplist的例子：
===========================
一下是一个ziplist包含两个元素分别是string“2”和string“5”，总共消耗15个字节。我们在视觉上将他划分为一下几个部分：
 *  [0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
 *        |             |          |       |       |     |
 *     zlbytes        zltail    entries   "2"     "5"   end

前四个字节代表数字15，他代表整个ziplist的长度，接下来四个字节代表最最后一个entry的偏移量，也就是12,最后一个entry也就是“5”，位于ziplist内偏移12字节的位置。接下来的16位代表代表ziplist的元素数量，也就是2.最后，[00 f3] 的代表数字2，它包含前一个entry的长度，也就是0，因为2就是第一个entry，同时btype F3编码|1111xxxx| xxxx的范围是0001 到 1101.我们需要去掉1111，并从3中减去1，因此这个entry就是2.下一个entry是的prevlen是2，表示前一项的长度是2 bytes. 他自己本身是F6，6-1=5，因此这个entry就是5.最终FF表示结尾.

向上面的字符串添加另一个元素，其值为“Hello World”允许我们展示 ziplist 如何编码小字符串。我们只展示条目本身的十六进制转储。想象一下上面的 ziplist 中存储“5”的条目后面的字节：

[02] [0b] [48 65 6c 6c 6f 20 57 6f 72 6c 64]

02代表prevlen，前一个entry的长度.第二个byte [0b] 代表编码形式|00pppppp|，长度也就是11，随后就是ASCII的“hello world”。
```
以上我们大致能够了解到ziplist的存储结构。

# 看看code
## 创建一个ziplist
```c
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
```
1. `zlbytes`赋值为`(sizeof(uint32_t)*2+sizeof(uint16_t))` + `(sizeof(uint8_t))`, intrev32ifbe是用来保持小端序的。
2. `zltail`赋值为`ZIPLIST_HEADER_SIZE`。
3. `zllen`赋值为0
4. `zlend` 赋值为`ZIP_END`， 也就是255。
也就是给ziplist赋了的个初值。


## insert entry
这个insert的逻辑偏复杂，我们一段段代码的来分析。首先插入方法`ziplistInsert`最终调用的是`__ziplistInsert`。所以我们主要来看看`__ziplistInsert`。
```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;

    /* Find out prevlen for the entry that is inserted. */
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLength(ptail);
        }
    }
    //.......
```

我们分析一下最开始的逻辑。
- 如果`p[0]`不是尾节点，则需要赋值将`p[0]`的`prevlen` 和 `prevlensize`赋值给__ziplistInsert中的`prevlensize, prevlen`，具体逻辑。
```c
// 这里我们可以看到如开头的注释所说，prevlensize是一个字节的话，prevlen直接读`(ptr)[0]`，否则读`ptr`的1-5个字节，同时保持大端序。
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
        (prevlensize) = 1;                                                     \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0);

#define ZIP_DECODE_PREVLEN(ptr, prevlensize, prevlen) do {                     \
    ZIP_DECODE_PREVLENSIZE(ptr, prevlensize);                                  \
    if ((prevlensize) == 1) {                                                  \
        (prevlen) = (ptr)[0];                                                  \
    } else if ((prevlensize) == 5) {                                           \
        assert(sizeof((prevlen)) == 4);                                    \
        memcpy(&(prevlen), ((char*)(ptr)) + 1, 4);                             \
        memrev32ifbe(&prevlen);                                                \
    }                                                                          \
} while(0);
```
- 如果`p[0]`就是结尾的话，else看着像进行了一些特殊处理。如果ziplist为空的话，`ptail`就是尾节点, 如果不为空的话，则进入if分支，此时p就成为了s的前置节点。也就是需要求p的 `<prevlen> <encoding> <entry-data>`的字节长度。
```C
#define ZIPLIST_ENTRY_TAIL(zl) ((zl)+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))

/* Return the total number of bytes used by the entry pointed to by 'p'. */
unsigned int zipRawEntryLength(unsigned char *p) {
    unsigned int prevlensize, encoding, lensize, len;
    ZIP_DECODE_PREVLENSIZE(p, prevlensize);
    ZIP_DECODE_LENGTH(p + prevlensize, encoding, lensize, len);
    return prevlensize + lensize + len;
}

#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
        (prevlensize) = 1;                                                     \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0);

#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len) do {                    \
    ZIP_ENTRY_ENCODING((ptr), (encoding));                                     \
    if ((encoding) < ZIP_STR_MASK) {                                           \
        if ((encoding) == ZIP_STR_06B) {                                       \
            (lensize) = 1;                                                     \
            (len) = (ptr)[0] & 0x3f;                                           \
        } else if ((encoding) == ZIP_STR_14B) {                                \
            (lensize) = 2;                                                     \
            (len) = (((ptr)[0] & 0x3f) << 8) | (ptr)[1];                       \
        } else if ((encoding) == ZIP_STR_32B) {                                \
            (lensize) = 5;                                                     \
            (len) = ((ptr)[1] << 24) |                                         \
                    ((ptr)[2] << 16) |                                         \
                    ((ptr)[3] <<  8) |                                         \
                    ((ptr)[4]);                                                \
        } else {                                                               \
            panic("Invalid string encoding 0x%02X", (encoding));               \
        }                                                                      \
    } else {                                                                   \
        (lensize) = 1;                                                         \
        (len) = zipIntSize(encoding);                                          \
    }                                                                          \
} while(0);

#define ZIP_ENTRY_ENCODING(ptr, encoding) do {  \
    (encoding) = (ptr[0]); \
    if ((encoding) < ZIP_STR_MASK) (encoding) &= ZIP_STR_MASK; \
} while(0)
```
- `prevlensize`用来保存p的prevlen的长度，这个逻辑主要在`ZIP_DECODE_PREVLENSIZE`中，其实就是p的第一个字节是小于254，则是一个字节存储，否则五个字节存储。
- 接着就是`ZIP_DECODE_LENGTH`其首先通过`ZIP_ENTRY_ENCODING`来确定encoding是否小于`ZIP_STR_MASK(11000000)`, 如果是则encoding就用来保存ptr的编码类型。
- 回到`ZIP_DECODE_LENGTH`，如果encoding < ZIP_STR_MASK代表是字符编码类型，则可以看是属于三种字符编码中的哪一种类型了。lensize保存`<encoding>`长度， len保存`<entry-data>`长度。
	- 如果是`ZIP_STR_06B`，则lensize = 1， len=prt[0] & 00111111，也就是第一个字节的后6为表示字符长度。
	- 如果是`ZIP_STR_14B`，则lensize = 2， len=(prt[0] & 00111111)<<8 | prt[1], 也就是前两个字节的后14位表示长度，
	- 如果是`ZIP_STR_32B`，则lensize = 5，len则是后四个字节，按照大端存储读取即可。
	- 其他就是int类型了，lensize=1，具体保存信息的长度则通过zipIntSize获得。这个看最开始的注解就懂了，记得别忘了`立即 4 位整数`。
- 最后就返回`prevlensize` + `lensize` + `len`则是 insert节点`s`的prevlen。

接下来进入第二段逻辑。
```c
if (zipTryEncoding(s,slen,&value,&encoding)) {
	/* 'encoding' is set to the appropriate integer encoding */
	reqlen = zipIntSize(encoding);
} else {
	/* 'encoding' is untouched, however zipStoreEntryEncoding will use the
	 * string length to figure out how to encode it. */
	reqlen = slen;
}

int zipTryEncoding(unsigned char *entry, unsigned int entrylen, long long *v, unsigned char *encoding) {
    long long value;

    if (entrylen >= 32 || entrylen == 0) return 0;
    if (string2ll((char*)entry,entrylen,&value)) {
        /* Great, the string can be encoded. Check what's the smallest
         * of our encoding types that can hold this value. */
        if (value >= 0 && value <= 12) {
            *encoding = ZIP_INT_IMM_MIN+value;
        } else if (value >= INT8_MIN && value <= INT8_MAX) {
            *encoding = ZIP_INT_8B;
        } else if (value >= INT16_MIN && value <= INT16_MAX) {
            *encoding = ZIP_INT_16B;
        } else if (value >= INT24_MIN && value <= INT24_MAX) {
            *encoding = ZIP_INT_24B;
        } else if (value >= INT32_MIN && value <= INT32_MAX) {
            *encoding = ZIP_INT_32B;
        } else {
            *encoding = ZIP_INT_64B;
        }
        *v = value;
        return 1;
    }
    return 0;
}

unsigned int zipIntSize(unsigned char encoding) {
    switch(encoding) {
    case ZIP_INT_8B:  return 1;
    case ZIP_INT_16B: return 2;
    case ZIP_INT_24B: return 3;
    case ZIP_INT_32B: return 4;
    case ZIP_INT_64B: return 8;
    }
    if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX)
        return 0; /* 4 bit immediate */
    panic("Invalid integer encoding 0x%02X", encoding);
    return 0;
}

unsigned char len = 1, buf[5];
unsigned int zipStoreEntryEncoding(unsigned char *p, unsigned char encoding, unsigned int rawlen) {

    if (ZIP_IS_STR(encoding)) {
        /* Although encoding is given it may not be set for strings,
         * so we determine it here using the raw length. */
        if (rawlen <= 0x3f) {
            if (!p) return len;
            //......
        } else if (rawlen <= 0x3fff) {
            len += 1;
            if (!p) return len;
            //......
        } else {
            len += 4;
            if (!p) return len;
            //......
        }
    } else {
        /* Implies integer encoding, so length is always 1. */
        if (!p) return len;
        buf[0] = encoding;
    }

    //......
    return len;
}

int zipPrevLenByteDiff(unsigned char *p, unsigned int len) {
    unsigned int prevlensize;
    ZIP_DECODE_PREVLENSIZE(p, prevlensize);
    return zipStorePrevEntryLength(NULL, len) - prevlensize;
}
```
- 这里先尝试了对s进行了转整型的编码，首先是如果其长度`(entrylen >= 32 || entrylen == 0)`，就无法进行整型编码，不应该大于20就不行了码？
	- 调用string2ll, 将string转为int，值保存value内。接着就是将encoding赋值为各种编码类型。v=value，同时返回1。
	- 返回主逻辑的if分支后，将`replen`赋值为保存实际数据的长度，`zipIntSize`在前面已经解释过。
	- 否则就是一个字符串，`reqlen=slen`即可。到这里，reqlen等于的是保存实际数据所需要的长度，即`<entry-data>`所需要的字节数。
- 接着是求`prevlen`所需的字节数，这个逻辑较为简单，不做过多解释。
- 随后是求保存`encoding`所需字节数，len初值是赋为1，int类型的encoding都是1，接着就是看他是属于那种类型的string，并将实际encoding保存在buf中，因为在p为null的时候，buf是没用用的，所以我注释掉了这一部分。
- 到这里，s需要的字节数已经求出来了。但是还是需要判断一下，如果p不是尾节点的话，s就插到p前面了，有可能是p的prevlensize是要扩大的。具体逻辑在`zipPrevLenByteDiff`, 看p存储len作为prevlen字节数是否够。所以nextdiff也是需要的字节。

其后开始进入拓容逻辑。
```c
offset = p-zl; 
zl = ziplistResize(zl,curlen+reqlen+nextdiff); 
p = zl+offset;
```
1. 首先是获取p相对zl的偏移量。
2. 接着重新对zl分配内存，curlen是原来的长度，reqlen是s需要的长度，diff是p需要扩大的长度。
3. 接着重新获取p的位置。

```c
if (p[0] != ZIP_END) {
	/* Subtract one because of the ZIP_END bytes */
	memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

	/* Encode this entry's raw length in the next entry. */
	if (forcelarge)
		zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
	else
		zipStorePrevEntryLength(p+reqlen,reqlen);

	/* Update offset for tail */
	ZIPLIST_TAIL_OFFSET(zl) =
		intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

	/* When the tail contains more than one entry, we need to take
	 * "nextdiff" in account as well. Otherwise, a change in the
	 * size of prevlen doesn't have an effect on the *tail* offset. */
	zipEntry(p+reqlen, &tail);
	if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
		ZIPLIST_TAIL_OFFSET(zl) =
			intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
	}
} else {
	/* This element will be the new tail. */
	ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
}
```
- 如果是`P[0] != ZIP_END`，就要将`p-zltail`这一段移动到新内存。
	- 这个forcelarge看着像是修复了一个bug，不过目前没有找到相关issue。所以这段逻辑我们只能先跳过。
	- 接着就是对`zltail`的赋值，需要加上一个`reqlen`。
	- 接下来的这段了逻辑比较复杂，首先是将p的一些信息存到`tail`中，接着是看p是不是最后一个entry，如果不是的，`ZIPLIST_TAIL_OFFSET`需要加上diff。
```c
/* Return a struct with all information about an entry. */
void zipEntry(unsigned char *p, zlentry *e) {

    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    e->headersize = e->prevrawlensize + e->lensize;
    e->p = p;
}
```

- 如果`p[0] = ZIP_END` ,则仅需要`ZIPLIST_TAIL_OFFSET=p-zl，此时s就是最后一个entry`。

接下来的这段逻辑比较重要，因为ziplist受限于连锁更新，所带来的稳定性较差，所以后面页产生了一些替代品。
```
if (nextdiff != 0) {
	offset = p-zl;
	zl = __ziplistCascadeUpdate(zl,p+reqlen);
	p = zl+offset;
}
```
稍后来看看这个`__ziplistCascadeUpdate`方法。先看看最后的逻辑。

- 最后就是写入这个entry, p此时仍然指向原位置。
	- 所以先写入prevlen。
	- 再写入encoding。
	- 最后在把s的`entry data` copy进来。
	- 最长长度+1即可。
```c
p += zipStorePrevEntryLength(p,prevlen);
p += zipStoreEntryEncoding(p,encoding,slen);
if (ZIP_IS_STR(encoding)) {
	memcpy(p,s,slen);
} else {
	zipSaveInteger(p,value,encoding);
}
ZIPLIST_INCR_LENGTH(zl,1);
return zl;
```

## 连锁更新的问题
其主要逻辑集中在`__ziplistCascadeUpdate`。
```c
/* When an entry is inserted, we need to set the prevlen field of the next
 * entry to equal the length of the inserted entry. It can occur that this
 * length cannot be encoded in 1 byte and the next entry needs to be grow
 * a bit larger to hold the 5-byte encoded prevlen. This can be done for free,
 * because this only happens when an entry is already being inserted (which
 * causes a realloc and memmove). However, encoding the prevlen may require
 * that this entry is grown as well. This effect may cascade throughout
 * the ziplist when there are consecutive entries with a size close to
 * ZIP_BIG_PREVLEN, so we need to check that the prevlen can be encoded in
 * every consecutive entry.
 *
 * Note that this effect can also happen in reverse, where the bytes required
 * to encode the prevlen field can shrink. This effect is deliberately ignored,
 * because it can cause a "flapping" effect where a chain prevlen fields is
 * first grown and then shrunk again after consecutive inserts. Rather, the
 * field is allowed to stay larger than necessary, because a large prevlen
 * field implies the ziplist is holding large entries anyway.
 *
 * The pointer "p" points to the first entry that does NOT need to be
 * updated, i.e. consecutive fields MAY need an update. */
```
首先我们来看看方法的注解。
插入`entry`时，我们需要将下一`entry`的 prevlen 字段设置为等于插入`entry`的长度。可能会出现此长度无法用 1 个字节编码的情况，因此下一个`entry`需要增大`prevlen`以容纳 5 个字节编码的 。这可以免费完成，因为这仅在已插入`entry`时发生（这会导致 realloc 和 memmove）。但是，编码 prevlen 可能还需要此条目也增大。当存在大小接近ZIP_BIG_PREVLEN 的连续条目时，此效果可能会在整个 ziplist 中级联，因此我们需要检查是否可以在每个连续条目中编码 prevlen。

请注意，此效果也可能反向发生，其中编码 prevlen 字段所需的字节数可能会减少。故意忽略此影响，因为它可能导致“抖动”效应，即链式 prevlen 字段首先增大，然后在连续插入后再次缩小。相反，允许字段保持大于必要的大小，因为较大的 prevlen字段意味着 ziplist 无论如何都会保存较大的条目。

指针“p”指向第一个不需要更新的条目，即连续字段可能需要更新。

接下来我们来看看函数。主要就是跑一个循环，我们直接进入循环开始看。
- 首先是获取存储p需要的字节长度`rawlen`和保存`rawlen`所需要的字节数。
```c
zipEntry(p, &cur);
rawlen = cur.headersize + cur.len;
rawlensize = zipStorePrevEntryLength(NULL,rawlen);

void zipEntry(unsigned char *p, zlentry *e) {

    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    e->headersize = e->prevrawlensize + e->lensize;
    e->p = p;
}
```

- 如果没有一一个节点，直接return。然后就可以通过zipentry函数解析下一个entry。还需要特判的是如果`next.prevrawlen == rawlen`则代表prevlen没有发生变化，无序处理。
```c
/* Abort if there is no next entry. */
if (p[rawlen] == ZIP_END) break;
zipEntry(p+rawlen, &next);

/* Abort when "prevlen" has not changed. */
if (next.prevrawlen == rawlen) break;
```

- 接着就是进行连锁更新的处理，首先是prevlen变大了。
	- 首先是先计算了一些有用的字段。`np`和`noffset`分别代表当前节点的位置和当前节点的偏移量。
	- 接着看如果当前节点不是最后一个节点，`zltail`需要+`extra`，即np额外需要的字节数。
	- 接着就是相当于将np向后移动extra个字节，
```C
/* The "prevlen" field of "next" needs more bytes to hold
 * the raw length of "cur". */
offset = p-zl;
extra = rawlensize-next.prevrawlensize;
zl = ziplistResize(zl,curlen+extra);
p = zl+offset;

/* Current pointer and offset for next element. */
np = p+rawlen;
noffset = np-zl;

/* Update tail offset when next element is not the tail element. */
if ((zl+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))) != np) {
	ZIPLIST_TAIL_OFFSET(zl) =
		intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
}

/* Move the tail to the back. */
memmove(np+rawlensize,
	np+next.prevrawlensize,
	curlen-noffset-next.prevrawlensize-1);
zipStorePrevEntryLength(np,rawlen);

/* Advance the cursor */
p += rawlen;
curlen += extra;
```

- 另外一种情况就是prevlen变小了。这种情况空间不发生变化，所以仅存储一下即可。需要注意的长度本来为5就不收缩了。也就是第一个if的特判。


以上就大概是ziplist的解析。后续看看redis针对ziplist的优化。



