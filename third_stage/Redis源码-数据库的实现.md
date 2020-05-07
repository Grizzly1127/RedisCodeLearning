# redisDb数据库

## 简介

---
源码位置：db.c/server.h

在前两个阶段中，我们学习了redis数据结构的实现，而这些数据结构都是为了实现数据库功能做的铺垫，下面，让我们一起来看看redis数据库是如何实现的吧。  

Redis服务器在运行的时候会创建大量的redisObject对象，这些对象都是存在redisDb中的，为了快速索引到某个对象，redisDb采用了dict字典结构设计。  
启动Redis后，Redis服务器将所有数据库都保存在redisServer结构的db数组中，db数组的每一项都是一个redisDb结构，代表一个数据库。根据配置参数，redis服务器在初始化的时候，会创建16个数据库，由dbnum决定(不可修改)。  
![redisServer](../img/stage3/db1.png)

下面我们通过代码来看看redisServer和redisDb的结构。

</br>
</br>

## 结构体与宏定义

---

``` c
// redisServer
struct redisServer {
    // ...
    redisDb *db;    /* 数据库数组*/
    // ...
    int dbnum;      /* 数据库的总个数 */
    // ...
};

// redisDb
typedef struct redisDb {
    dict *dict;                 /* 数据库的键空间 */
    dict *expires;              /* 键过期时间 */
    dict *blocking_keys;        /* 存放所有造成阻塞的键及其客户端 */
    dict *ready_keys;           /* 存放push操作添加的造成阻塞的键，便于解阻塞 */
    dict *watched_keys;         /* 被watch命令监控的键和相应的客户端，用于multi/exec */
    int id;                     /* 数据库编号 */
    long long avg_ttl;          /* 平均生存时间 */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* 要尝试逐项进行碎片整理的密钥名称列表 */
} redisDb;
```

</br>

## 函数功能总览

---

``` c
int removeExpire(redisDb *db, robj *key); // 移除key的过期时间，当只有在db->dict中存在key时，才会移除
void propagateExpire(redisDb *db, robj *key, int lazy);
int expireIfNeeded(redisDb *db, robj *key);
long long getExpire(redisDb *db, robj *key);
void setExpire(client *c, redisDb *db, robj *key, long long when);
robj *lookupKey(redisDb *db, robj *key, int flags);
robj *lookupKeyRead(redisDb *db, robj *key);
robj *lookupKeyWrite(redisDb *db, robj *key);
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply);
robj *lookupKeyWriteOrReply(client *c, robj *key, robj *reply);
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags);
robj *lookupKeyWriteWithFlags(redisDb *db, robj *key, int flags);
robj *objectCommandLookup(client *c, robj *key);
robj *objectCommandLookupOrReply(client *c, robj *key, robj *reply);
int objectSetLRUOrLFU(robj *val, long long lfu_freq, long long lru_idle,
                       long long lru_clock, int lru_multiplier);
#define LOOKUP_NONE 0
#define LOOKUP_NOTOUCH (1<<0)
void dbAdd(redisDb *db, robj *key, robj *val);
void dbOverwrite(redisDb *db, robj *key, robj *val);
void genericSetKey(redisDb *db, robj *key, robj *val, int keepttl);
void setKey(redisDb *db, robj *key, robj *val);
int dbExists(redisDb *db, robj *key);
robj *dbRandomKey(redisDb *db);
int dbSyncDelete(redisDb *db, robj *key);
int dbDelete(redisDb *db, robj *key);
robj *dbUnshareStringValue(redisDb *db, robj *key, robj *o);

#define EMPTYDB_NO_FLAGS 0      /* No flags. */
#define EMPTYDB_ASYNC (1<<0)    /* Reclaim memory in another thread. */
#define EMPTYDB_BACKUP (1<<2)   /* DB array is a backup for REPL_DISKLESS_LOAD_SWAPDB. */
long long emptyDb(int dbnum, int flags, void(callback)(void*));
long long emptyDbGeneric(redisDb *dbarray, int dbnum, int flags, void(callback)(void*));
void flushAllDataAndResetRDB(int flags);
long long dbTotalServerKeyCount();

int selectDb(client *c, int id);
void signalModifiedKey(redisDb *db, robj *key);
void signalFlushedDb(int dbid);
unsigned int getKeysInSlot(unsigned int hashslot, robj **keys, unsigned int count);
unsigned int countKeysInSlot(unsigned int hashslot);
unsigned int delKeysInSlot(unsigned int hashslot);
int verifyClusterConfigWithData(void);
void scanGenericCommand(client *c, robj *o, unsigned long cursor);
int parseScanCursorOrReply(client *c, robj *o, unsigned long *cursor);
void slotToKeyAdd(robj *key);
void slotToKeyDel(robj *key);
void slotToKeyFlush(void);
int dbAsyncDelete(redisDb *db, robj *key);
void emptyDbAsync(redisDb *db);
void slotToKeyFlushAsync(void);
size_t lazyfreeGetPendingObjectsCount(void);
void freeObjAsync(robj *o);

/* API to get key arguments from commands */
int *getKeysFromCommand(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
void getKeysFreeResult(int *result);
int *zunionInterGetKeys(struct redisCommand *cmd,robj **argv, int argc, int *numkeys);
int *evalGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
int *sortGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
int *migrateGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
int *georadiusGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
int *xreadGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
int *memoryGetKeys(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);
```
