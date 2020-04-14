# stream消息队列

## 简介

---
源码位置：t_stream.c/stream.h

官方描述：Stream是Redis 5.0版本引入的一个新的数据类型，它以更抽象的方式模拟日志数据结构，但日志仍然是完整的：就像一个日志文件，通常实现为以只附加模式打开的文件，Redis流主要是一个仅附加数据结构。[点击查看官方介绍](http://www.redis.cn/topics/streams-intro.html)
</br>
</br>

## 结构体与宏定义

---

``` c
typedef struct streamID {
    uint64_t ms;            /* unix时间戳（毫秒） */
    uint64_t seq;           /* 序列号 */
} streamID;

typedef struct stream {
    rax *rax;               /* 核心数据结构基数树 */
    uint64_t length;        /* stream中的元素个数 */
    streamID last_id;       /* 最新的stream id. */
    rax *cgroups;           /* 消费者组字典 */
} stream;

typedef struct streamIterator {
    stream *stream;         /* The stream we are iterating. */
    streamID master_id;     /* ID of the master entry at listpack head. */
    uint64_t master_fields_count;       /* Master entries # of fields. */
    unsigned char *master_fields_start; /* Master entries start in listpack. */
    unsigned char *master_fields_ptr;   /* Master field to emit next. */
    int entry_flags;                    /* Flags of entry we are emitting. */
    int rev;                /* True if iterating end to start (reverse). */
    uint64_t start_key[2];  /* Start key as 128 bit big endian. */
    uint64_t end_key[2];    /* End key as 128 bit big endian. */
    raxIterator ri;         /* Rax iterator. */
    unsigned char *lp;      /* Current listpack. */
    unsigned char *lp_ele;  /* Current listpack cursor. */
    unsigned char *lp_flags; /* Current entry flags pointer. */
    /* Buffers used to hold the string of lpGet() when the element is
     * integer encoded, so that there is no string representation of the
     * element inside the listpack itself. */
    unsigned char field_buf[LP_INTBUF_SIZE];
    unsigned char value_buf[LP_INTBUF_SIZE];
} streamIterator;

typedef struct streamCG {
    streamID last_id;
    rax *pel;
    rax *consumers;
} streamCG;

typedef struct streamConsumer {
    mstime_t seen_time;
    sds name;
    rax *pel;
} streamConsumer;

/* Pending (yet not acknowledged) message in a consumer group. */
typedef struct streamNACK {
    mstime_t delivery_time;
    uint64_t delivery_count;
    streamConsumer *consumer;
} streamNACK;

/* Stream propagation informations, passed to functions in order to propagate
 * XCLAIM commands to AOF and slaves. */
typedef struct streamPropInfo {
    robj *keyname;
    robj *groupname;
} streamPropInfo;
```
