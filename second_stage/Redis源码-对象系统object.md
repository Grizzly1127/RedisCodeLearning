# object对象系统

## 简介

---
源码位置：object.c/server.h

在之前的文章中，我们介绍了redis底层的数据结构，比如简单动态字符串，双端链表，跳跃表，字典，整数集合，压缩列表，快速列表，基数树，紧凑列表等。  
然而Redis没有用这些数据结构来实现键值对的数据库，而是在这些数据结构之上又封装了一层RedisObject，RedisObject有6种类型：string字符串，hash散列，set集合，zset有序集合，list列表，stream消息队列这些类型是面向用户的，有些对象内部至少有两种编码方式，不同的编码方式适用于不同的使用场景。  
Redis对象带有引用计数功能，类似于智能指针，当引用计数为0时，对象将会被自动释放。
Redis还会对每一个对象记录其最近被使用时间，从而计算对象的空转时长，便于在适当的时候释放内存。

redis对象的类型和其对应使用的编码方式（数据结构）：  
|type|encoding|
|---|---|
|OBJ_STRING|OBJ_ENCODING_RAW ,OBJ_ENCODING_INT ,OBJ_ENCODING_EMBSTR|
|OBJ_LIST|OBJ_ENCODING_QUICKLIST|
|OBJ_SET|OBJ_ENCODING_INTSET ,OBJ_ENCODING_HT|
|OBJ_ZSET|OBJ_ENCODING_ZIPLIST ,OBJ_ENCODING_SKIPLIST|
|OBJ_HASH|OBJ_ENCODING_ZIPLIST ,OBJ_ENCODING_HT|
|OBJ_STREAM|OBJ_ENCODING_STREAM|

</br>
</br>

## 结构体与宏定义

---

``` c
#define LRU_BITS 24

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
/*
type：标识redis对象的七种类型，占用4个位
encoding：标识redis对象的编码方式，也就是ptr所指向的数据用何种数据结构作为底层实现方式，占用4个位
lru:最后一次被访问的时间，占用24个位
refcount：引用计数，实现自动内存回收机制。
    1. 当创建一个对象时，其引用计数初始化为1；
    2. 当这个对象被一个新程序使用时，其引用计数加1；
    3. 当这个对象不再被一个程序使用时，其引用计数减1；
    4. 当引用计数为0时，释放该对象，回收内存。
ptr：指向真正的存储结构
*/

/* 七种对象类型 */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */

/* 11种对象编码类型，不同的对象类型会根据实际情况选择不同的编码类型 */
#define OBJ_ENCODING_RAW 0     /* Raw representation */ // sds简单动态字符串
#define OBJ_ENCODING_INT 1     /* Encoded as integer */ // long类型的整数
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */ // dict字典
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */ // 不再使用
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */ // adlist双端链表
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */ // ziplist压缩列表
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */ // intset整数集合
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */ // skiplist跳跃表
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */ // EMBSTR编码的简单动态字符串sds
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */ // quicklist快速列表
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */ // stream消息队列

// 当字符串小于44字节时，使用EMBSTR编码
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
```

</br>

## 函数功能总览

``` c
/* Redis object implementation */
void decrRefCount(robj *o); // 引用计数减1
void decrRefCountVoid(void *o); // 调用decrRefCount
void incrRefCount(robj *o); // 引用计数加1
robj *makeObjectShared(robj *o); // 共享对象，将引用计数设定为一个特殊值OBJ_SHARED_REFCOUNT(INT_MAX)
robj *resetRefCount(robj *obj); // 将引用计数置为0
// free object-------------
void freeStringObject(robj *o); // 释放字符串对象
void freeListObject(robj *o); // 释放列表对象
void freeSetObject(robj *o); // 释放集合对象
void freeZsetObject(robj *o); // 释放有序集合对象
void freeHashObject(robj *o); // 释放哈希对象
// ------------------------
// create object-----------
robj *createObject(int type, void *ptr); // 创建对象，设定类型
robj *createStringObject(const char *ptr, size_t len); // 创建字符串对象
robj *createRawStringObject(const char *ptr, size_t len); // 创建SDS的字符串对象
robj *createEmbeddedStringObject(const char *ptr, size_t len); // 创建EMBSTR编码的字符串对象
// ------------------------
robj *dupStringObject(const robj *o); // 复制string对象
int isSdsRepresentableAsLongLong(sds s, long long *llval); // sds转为long long
int isObjectRepresentableAsLongLong(robj *o, long long *llongval); // 判断一个对象是否能用long long表示
robj *tryObjectEncoding(robj *o); // 尝试对字符串对象进行编码，以节省空间，如果无法压缩，则增加引用计数
robj *getDecodedObject(robj *o); // 获取字符串编码对象的解码版本，能解码则返回一个新的对象，不能则增加引用计数
size_t stringObjectLen(robj *o); // 获取字符串对象的长度
// create object-----------
robj *createStringObjectFromLongLong(long long value); // 根据传入的longlong整型值，创建一个字符串对象
robj *createStringObjectFromLongLongForValue(long long value);
robj *createStringObjectFromLongDouble(long double value, int humanfriendly); // 根据传入的long double类型值，创建一个字符串对象
robj *createQuicklistObject(void); // 创建快速列表对象
robj *createZiplistObject(void); // 创建压缩列表对象
robj *createSetObject(void); // 创建集合对象
robj *createIntsetObject(void); // 创建整数集合对象
robj *createHashObject(void); // 创建hash对象
robj *createZsetObject(void); // 创建有序集合对象
robj *createZsetZiplistObject(void); // 创建压缩列表编码的有序集合对象
robj *createStreamObject(void); // 创建消息队列对象
robj *createModuleObject(moduleType *mt, void *value); // 创建模块对象
// -------------------------
int getLongFromObjectOrReply(client *c, robj *o, long *target, const char *msg); // getLongLongFromObject函数的封装，如果发生错误可以发回指定响应消息
int checkType(client *c, robj *o, int type); // 检查o的类型是否与type一致
int getLongLongFromObjectOrReply(client *c, robj *o, long long *target, const char *msg); // getLongLongFromObject的封装，如果发生错误则可以发出指定的错误消息
int getDoubleFromObjectOrReply(client *c, robj *o, double *target, const char *msg); // getDoubleFromObject的封装，如果发生错误可以发回指定响应消息
int getDoubleFromObject(const robj *o, double *target); // 从字符串对象中解码出一个double类型的整数
int getLongLongFromObject(robj *o, long long *target); // 从字符串对象中解码出一个long long类型的整数
int getLongDoubleFromObject(robj *o, long double *target); // 从字符串对象中解码出一个long double类型的整数
int getLongDoubleFromObjectOrReply(client *c, robj *o, long double *target, const char *msg); // getLongDoubleFromObject的封装，如果发生错误可以发回指定响应消息
char *strEncoding(int encoding); // 返回编码对应的字符串名称
int compareStringObjects(robj *a, robj *b); // 以二进制方式比较两个字符串对象
int collateStringObjects(robj *a, robj *b); // 以本地指定的文字排列次序coll方式比较两个字符串
int equalStringObjects(robj *a, robj *b); // 比较两个字符串对象是否相同
unsigned long long estimateObjectIdleTime(robj *o); // 计算给定对象的闲置时长，使用近似LRU算法
void trimStringObjectIfNeeded(robj *o); // 优化字符串对象中的SDS字符串
#define sdsEncodedObject(objptr) (objptr->encoding == OBJ_ENCODING_RAW || objptr->encoding == OBJ_ENCODING_EMBSTR)
```
