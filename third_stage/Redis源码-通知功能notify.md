# notify通知功能

## 简介

---
源码位置：notify.c/server.h

**1. 事件通知概述**
对于Redis服务器，它可以通过订阅发布功能来发送服务器中的键空间事件。所谓的键空间事件，就是数据库中键的增加、修改和删除等操作，用于告知收听该类事件的客户端当前数据库中执行了哪些操作。客户端可以通过 订阅与发布功能（pub/sub）功能，来接收那些以某种方式改动了Redis数据集的事件。  
目前Redis的订阅与发布功能采用的是发送即忘（fire and forget）的策略，当订阅事件的客户端断线时，它会丢失所有在断线期间分发给它的事件。  
订阅与发布功能（pub/sub）功能实现在pubsub.c中，后续会在博文中讲到。

**2. 事件类型**
对于每个修改数据库的操作，键空间通知都会发送两种不同类型的事件：  

* 键空间通知（key-space）
* 键事件通知（key-event）

当 `del mykey` 命令执行时：

* 键空间频道的订阅者将接收到被执行的事件的名字，在这个例子中，就是 del
* 键事件频道的订阅者将接收到被执行事件的键的名字，在这个例子中，就是 mykey

**3. 配置**
因为开启键空间通知功能需要消耗一些 CPU，所以在默认配置下，该功能处于关闭状态。开启通知功能有以下两种方式：  

1. 修改 redis.conf 中的 `notify-keyspace-events` 参数
2. 通过 CONFIG SET 命令来设定 `notify-keyspace-events` 参数

`notify-keyspace-events` 参数可以是以下字符的任意组合， 它指定了服务器该发送哪些类型的通知：  
|字符|发送的通知|
|---|---|
|K|键空间通知，所有通知以 __keyspace@\<db>__ 为前缀|
|E|键事件通知，所有通知以 __keyevent@\<db>__ 为前缀|
|g|DEL 、 EXPIRE 、 RENAME 等类型无关的通用命令的通知|
|$|字符串命令的通知|
|l|列表命令的通知|
|s|集合命令的通知|
|h|哈希命令的通知|
|z|有序集合命令的通知|
|x|过期事件：每当有过期键被删除时发送|
|e|驱逐(evict)事件：每当有键因为 maxmemory 政策而被删除时发送|
|A|参数 g$lshzxe 的别名，包含所有的字符|

输入的参数中至少要有一个 K 或者 E，否则的话，不管其余的参数是什么，都不会有任何通知被分发。
在源码中设定了一系列的宏定义，用来标识以上这些字符事件的类型：

```c
// 键空间通知的类型，每个类型都关联着一个有目的的字符
#define NOTIFY_KEYSPACE (1<<0)    /* K */
#define NOTIFY_KEYEVENT (1<<1)    /* E */
#define NOTIFY_GENERIC (1<<2)     /* g */
#define NOTIFY_STRING (1<<3)      /* $ */
#define NOTIFY_LIST (1<<4)        /* l */
#define NOTIFY_SET (1<<5)         /* s */
#define NOTIFY_HASH (1<<6)        /* h */
#define NOTIFY_ZSET (1<<7)        /* z */
#define NOTIFY_EXPIRED (1<<8)     /* x */
#define NOTIFY_EVICTED (1<<9)     /* e */
#define NOTIFY_STREAM (1<<10)     /* t */
#define NOTIFY_KEY_MISS (1<<11)   /* m (Note: This one is excluded from NOTIFY_ALL on purpose) */
#define NOTIFY_ALL (NOTIFY_GENERIC | NOTIFY_STRING | NOTIFY_LIST | NOTIFY_SET | NOTIFY_HASH | NOTIFY_ZSET | NOTIFY_EXPIRED | NOTIFY_EVICTED | NOTIFY_STREAM) /* A flag */
```

在notify.c文件中，只有三个功能函数，下面让我们来看看源码实现吧。

</br>

## 函数功能总览

---

```c
/* Keyspace events notification */
void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid);
int keyspaceEventsStringToFlags(char *classes);
sds keyspaceEventsFlagsToString(int flags);
```

</br>

## 主要函数实现

---

```c
/* 因为redis命令中事件类型是字符类型，所以会使用一个int类型的flags参数
 * 通过多个字符按位或运算保存起来，方便后面使用 */
int keyspaceEventsStringToFlags(char *classes) {
    char *p = classes;
    int c, flags = 0;

    while((c = *p++) != '\0') {
        switch(c) {
        case 'A': flags |= NOTIFY_ALL; break;
        case 'g': flags |= NOTIFY_GENERIC; break;
        case '$': flags |= NOTIFY_STRING; break;
        case 'l': flags |= NOTIFY_LIST; break;
        case 's': flags |= NOTIFY_SET; break;
        case 'h': flags |= NOTIFY_HASH; break;
        case 'z': flags |= NOTIFY_ZSET; break;
        case 'x': flags |= NOTIFY_EXPIRED; break;
        case 'e': flags |= NOTIFY_EVICTED; break;
        case 'K': flags |= NOTIFY_KEYSPACE; break;
        case 'E': flags |= NOTIFY_KEYEVENT; break;
        case 't': flags |= NOTIFY_STREAM; break;
        case 'm': flags |= NOTIFY_KEY_MISS; break;
        default: return -1;
        }
    }
    return flags;
}
```

```c
// 将flags参数转为sds类型
sds keyspaceEventsFlagsToString(int flags) {
    sds res;

    res = sdsempty();
    if ((flags & NOTIFY_ALL) == NOTIFY_ALL) {
        res = sdscatlen(res,"A",1);
    } else {
        if (flags & NOTIFY_GENERIC) res = sdscatlen(res,"g",1);
        if (flags & NOTIFY_STRING) res = sdscatlen(res,"$",1);
        if (flags & NOTIFY_LIST) res = sdscatlen(res,"l",1);
        if (flags & NOTIFY_SET) res = sdscatlen(res,"s",1);
        if (flags & NOTIFY_HASH) res = sdscatlen(res,"h",1);
        if (flags & NOTIFY_ZSET) res = sdscatlen(res,"z",1);
        if (flags & NOTIFY_EXPIRED) res = sdscatlen(res,"x",1);
        if (flags & NOTIFY_EVICTED) res = sdscatlen(res,"e",1);
        if (flags & NOTIFY_STREAM) res = sdscatlen(res,"t",1);
    }
    if (flags & NOTIFY_KEYSPACE) res = sdscatlen(res,"K",1);
    if (flags & NOTIFY_KEYEVENT) res = sdscatlen(res,"E",1);
    if (flags & NOTIFY_KEY_MISS) res = sdscatlen(res,"m",1);
    return res;
```

```c
// 利用Redis的订阅和发布功能来发送键空间事件通知。
void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid) {
    sds chan;
    robj *chanobj, *eventobj;
    int len = -1;
    char buf[24];

    /* 如果任何模块对事件感兴趣，将立即通知感兴趣的模块。这将绕过通知配置，但模块引擎将仅在事件类型与事件订阅者感兴趣的类型匹配时调用事件订阅者。 */
     moduleNotifyKeyspaceEvent(type, event, key, dbid);

    /* 通知功能关闭，直接退出 */
    if (!(server.notify_keyspace_events & type)) return;

    // 创建事件对象
    eventobj = createStringObject(event,strlen(event));

    /* 键空间通知，格式为__keyspace@<db>__:<key> <event> */
    if (server.notify_keyspace_events & NOTIFY_KEYSPACE) {
        chan = sdsnewlen("__keyspace@",11);
        len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, key->ptr);
        chanobj = createObject(OBJ_STRING, chan);
        // 调用pub/sub命令发送
        pubsubPublishMessage(chanobj, eventobj);
        decrRefCount(chanobj);
    }

    /* 键时间通知，格式为__keyevente@<db>__:<event> <key> */
    if (server.notify_keyspace_events & NOTIFY_KEYEVENT) {
        chan = sdsnewlen("__keyevent@",11);
        if (len == -1) len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, eventobj->ptr);
        chanobj = createObject(OBJ_STRING, chan);
        // 调用pub/sub命令发送
        pubsubPublishMessage(chanobj, key);
        decrRefCount(chanobj);
    }
    decrRefCount(eventobj);
}
```
