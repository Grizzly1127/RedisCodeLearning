# 事件驱动

## 简介

---
源码位置：ae.c/ae.h/ae_evport.c/ae_epoll.c/ae_kqueue.c/ae_select.c

**1. 简介**
Redis 是一个事件驱动的内存数据库，服务器需要处理两种类型的事件：

- 文本（IO）事件`AE_FILE_EVENTS`
- 时间事件`AE_TIME_EVENTS`

**2. 文本（IO）事件**
Redis是基于 Reactor 模式开发了自己的网络事件处理器，这个处理器被称为文件事件处理器。

- 文件事件处理器使用 I/O 多路复用（multiplexing）模型来同时监听多个套接字， 并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
- 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时， 与操作相对应的文件事件就会产生， 这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。

虽然文件事件处理器以单线程方式运行， 但通过使用 I/O 多路复用模型来监听多个套接字， 文件事件处理器既实现了高性能的网络通信模型， 又可以很好地与 Redis 服务器中其他同样以单线程方式运行的模块进行对接， 这保持了 Redis 内部单线程设计的简单性。
优势：

- 完全基于内存，绝大部分请求是纯粹的内存操作，非常快速
- 数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的
- 采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作。
- 使用多路I/O复用模型，非阻塞IO。

Redis提供了4中 I/O 多路复用的方式，其性能由高到低依次是：`evport` -> `epoll` -> `kqueue` -> `select`。
不同的平台使用了不同的实现方式，比如 epoll 和 select 可以用于Linux平台，kqueue 用于MacOS平台，select 用于Windows平台，evport 用于Solaris平台。
代码如下：

```c
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

一般情况下，Redis服务端都是部署在Linux系统上的，所以本文内容就只解析Redis是怎么利用epoll实现 I/O 多路复用的吧。

**epoll：**
epoll相关的接口实现都封装在了ae_epoll.c中，主要提供了以下几个函数：

```c
static int aeApiCreate(aeEventLoop *eventLoop); // 调用epoll_create创建epoll实例
static int aeApiResize(aeEventLoop *eventLoop, int setsize); // 重新设置epoll_event的大小
static void aeApiFree(aeEventLoop *eventLoop); // 释放实例
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask); // 当有新的客户端连接时，将新的fd注册到epoll实例中
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask); // 当有客户端断开连接时，将epoll实例中该客户端的fd删除
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp); // 调用epoll_wait获取客户端产生的io事件
static char *aeApiName(void) { return "epoll"; }
```

在redis.h/redisServer 结构中，保存着基于事件驱动的程序状态：

```c
struct redisServer {
    // ...
    aeEventLoop *el;
    // ...
}

typedef struct aeEventLoop {
    int maxfd;   /* 当前注册的最大fd */
    int setsize; /* 最大fd数 */
    long long timeEventNextId;
    time_t lastTime;     /* 用于检测系统时钟偏差 */
    aeFileEvent *events; /* 注册事件 */
    aeFiredEvent *fired; /* 触发事件 */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* 用于轮询特定于API的数据 */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```

Redis服务器在启动时，首先会创建事件轮询器：

```c
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;

    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    eventLoop->timeEventHead = NULL;
    eventLoop->timeEventNextId = 0;
    eventLoop->stop = 0;
    eventLoop->maxfd = -1;
    eventLoop->beforesleep = NULL;
    eventLoop->aftersleep = NULL;
    eventLoop->flags = 0;
    // 根据不同平台调用不同的 I/O 多路复用模型
    if (aeApiCreate(eventLoop) == -1) goto err;
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;

err:
    if (eventLoop) {
        zfree(eventLoop->events);
        zfree(eventLoop->fired);
        zfree(eventLoop);
    }
    return NULL;
}
```

创建完成后，开始事件轮询主循环：

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

事件的调度和执行函数：

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* 如果不处理时间事件或文本时间，直接返回 */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* 注意，只要我们想处理时间事件，即使没有要处理的文件事件，我们也要调用select（），以便在下一个时间事件准备好触发之前休眠 */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            // 获取到达时间离当前时间最接近的时间事件
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;

            /* 计算剩余时间 */
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;

            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            /* 如果我们必须检查事件，但由于AE_DONT_WAIT而需要尽快返回，我们需要将超时设置为0 */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                tvp = NULL; /* wait forever */
            }
        }

        if (eventLoop->flags & AE_DONT_WAIT) {
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        }

        /* 等待事件触发或者超时 */
        numevents = aeApiPoll(eventLoop, tvp);

        /* 阻塞后处理函数 */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */

            /* 如果设置了AE_BARRIER标志，我们优先处理写事件 */
            int invert = fe->mask & AE_BARRIER;

            /* 处理读事件 */
            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }

            /* 处理写事件 */
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* 如果需要反转读写处理顺序，处理完写事件后，可以处理读事件 */
            if (invert && fe->mask & mask & AE_READABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }
    /* 处理时间事件 */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* 返回处理的文本/时间事件数 */
}
```

**3. 时间事件**
Redis有很多操作需要在给定的时间点进行处理，时间事件就是对这类定时任务的抽象。
先看看时间事件的数据结构：

```c
typedef struct aeTimeEvent {
    long long id; /* 时间事件唯一标识符 */
    /* 事件的到达时间 */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    /* 事件处理函数 */
    aeTimeProc *timeProc;
    /* 事件释放函数 */
    aeEventFinalizerProc *finalizerProc;
    /* 多路复用库的私有数据 */
    void *clientData;
    /* 指向前、后两个时间事件结构，形成双向链表 */
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
} aeTimeEvent;
```

处理时间事件的函数：

```c
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;
    time_t now = time(NULL);

    /* 1. 记录最新一次执行这个函数的时间，用于处理系统时间被修改产生的问题 */
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    eventLoop->lastTime = now;

    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    /* 2. 遍历链表找出所有 when_sec 和 when_ms 小于现在时间的事件。 */
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* 如果当前的事件被设置为删除，则删除该事件 */
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        aeGetTime(&now_sec, &now_ms);
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            /* 3. 执行事件对应的处理函数 */
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            /* 4. 检查事件类型，如果是周期事件则刷新该事件下一次的执行事件，否则从列表中删除事件 */
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
    return processed;
}
```

**4. 总结**
为什么单线程的Redis会这么快？

- Redis是完全基于内存的数据库，绝大部分请求是纯粹的内存操作，非常快速。
- Redis由C语言编写，数据结构简单，对数据的操作也简单，Redis中的数据结构是专门设计的。
- Redis采用单线程，保证了数据操作的原子性，不存在多进程或者多线程导致切换而消耗CPU，避免了不必要的上下文切换和竞争条件，不存在加/解锁的操作，不用考虑可能出现的死锁导致性能消耗。
- 使用 I/O 多路复用模型，非阻塞IO，可以处理并发的连接。
