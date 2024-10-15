---
title: redis(一) SDS
date: 2024-10-15 23:07:00
categories: 
- redis
---

接下来一系列的文章可能会看一看redis的源码，主要是为了学习下优秀软件的设计思路。
# 为什么需要SDS？
C语言没有像其他语言一样的string类，有原生的字符串。C语言可以用`char*`存储字符串，并以`"\0"`结尾。但是 `char*`原生的增 也就是`strcat`在char数组长度不够的时候会溢出，即不存在拓容机制。

以 `"\0"`结尾所带来的后果就是字符串中间无法存储`"\0"`，这可能会导致一些编解码数据无法存储，比如图片，视频之类的。

# 内存使用上的优化
SDS的结构体大概如下。
```c
struct __attribute__ ((__packed__)) sdshdr8 {
	uint8_t len; /* used */
	uint8_t alloc; /* excluding the header and null terminator */
	unsigned char flags; /* 3 lsb of type, 5 unused bits */
	char buf[];
};
```
以上是SDS的结构设计，len表示字符长度，alloc表示当前分配的空间长度，flags表示SDS的类型，最后buf保存的则是实际数据。

其还有`sdshdr16`, `sdshdr32`, `sdshdr64`, 主要的区别就在于len和alloc所占用的字节数。同时增加 `__attribute__ ((__packed__))`，其作用是告诉编译器，禁止使用内存对齐。默认情况下，编译器会按照 8 字节对齐的方式，给变量分配内存。

同时结构体中保存len，求长度的时候可以直接返回。


# 一些不错的优化
首先我们需要来看看创建SDS的方法。
```c
sds sdsnewlen(const void *init, size_t initlen) { 
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1);
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    s = (char*)sh+hdrlen; // char [] 在结构体中不占用空间
    fp = ((unsigned char*)s)-1;
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```
- 1.首先根据initlen确定type和type的内存占用字节
- 2.接着是sh分配空间，将s指向buf, fp指向flag
- 3.然后就是给sh，s， fp赋值
- 4.同时以"\0"结尾。
`SDS_HDR_VAR`的定义是`#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)))`，就是将sh转为对应的`sdshdr`类型。

## 拼接字符串
接着我们来看看SDS拼接字符串。
```c
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}
```
-  1.首先是看内存是否还够存储下这个字符串t
- 2.够的话就copy过去
- 3.然后set s 的新 len

这里需要再看看的`sdsMakeRoomFor`。这里我们需要看看他的拓容机制。
```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
    return s;
}
```
  - 1.首先获取s的剩余容量avail，oldtype(s[-1]表示的s的上一个字节，即sds->flag)
  - 2.先判断s的剩余容量是否够用，够用直接返回即可
  - 3.接着就是拓容机制， < SDS_MAX_PREALLOC 则 * 2，否则就直接+上SDS_MAX_PREALLOC 
  - 4.接着就是判断newsh的type是否需要变化
拓展的常见方案也就是取一个临界值，小于他则* 2，大于他则+上他，避免一直* 2扩容。SDS_MAX_PREALLOC的取值是1024 * 1024 也就是1MB的大小。

