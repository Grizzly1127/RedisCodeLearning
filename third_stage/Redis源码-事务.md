# 事务

## 简介

---
源码位置：multi.c/redis.h

**1. 简介**
Redis 事务的本质是一组命令的集合。事务支持一次执行多个命令，一个事务中所有命令都会被序列化。在事务执行过程，会按照顺序串行化执行队列中的命令，其他客户端提交的命令请求不会插入到事务执行命令序列中。
总结说：redis事务就是一次性、顺序性、排他性的执行一个队列中的一系列命令。

Redis事务的ACID特性：

**A原子性（atomicity）**
单个Redis命令的执行是原子性的，但Redis没有在事务上增加任何维护原子性的机制，所以**Redis事务的执行不是原子性**的。
另一方面，如果 Redis 服务器进程在执行事务的过程中被停止 —— 比如接到 KILL 信号、宿主机器停机，等等，那么事务执行失败
事务失败时，Redis 也不会进行任何的重试或者回滚动作，不满足要么全部全部执行，要么都不执行的条件

**C一致性（consistency）：**
一致性分下面几种情况来讨论：

首先，如果一个事务的指令全部被执行，那么数据库的状态是满足数据库完整性约束的

其次，如果一个事务中有的指令有错误，那么数据库的状态是满足数据完整性约束的

最后，如果事务运行到某条指令时，进程被kill掉了，那么要分下面几种情况讨论：

- 如果当前redis采用的是内存模式，那么重启之后redis数据库是空的，那么满足一致性条件
- 如果当前采用RDB模式存储的，在执行事务时，Redis 不会中断事务去执行保存 RDB 的工作，只有在事务执行之后，保存 RDB 的工作才有可能开始。所以当 RDB 模式下的 Redis 服务器进程在事务中途被杀死时，事务内执行的命令，不管成功了多少，都不会被保存到 RDB 文件里。 恢复数据库需要使用现有的 RDB 文件，而这个 RDB 文件的数据保存的是最近一次的数 据库快照（snapshot），所以它的数据可能不是最新的，但只要 RDB 文件本身没有因为其他问题而出错，那么还原后的数据库就是一致的
- 如果当前采用的是AOF存储的，那么可能事务的内容还未写入到AOF文件，那么此时肯定是满足一致性的，如果事务的内容有部分写入到AOF文件中，那么需要用工具把AOF中事务执行部分成功的指令移除，这时，移除之后的AOF文件也是满足一致性的

所以，**redis事务满足一致性约束**。

**I隔离性（isolation）：**
Redis 是单进程程序，并且它保证在执行事务时，不会对事务进行中断，事务可以运行直到执行完所有事务队列中的命令为止。因此，**Redis 的事务是总是带有隔离性**的。

**D持久性（durability）：**
因为事务不过是用队列包裹起了一组 Redis 命令，并没有提供任何额外的持久性功能，所以事务的持久性由 Redis 所使用的持久化模式决定

- 在单纯的内存模式下，事务肯定是不持久的
- 在 RDB 模式下，服务器可能在事务执行之后、RDB 文件更新之前的这段时间失败，所以 RDB 模式下的 Redis 事务也是不持久的
- 在 AOF 的“总是 SYNC ”模式下，事务的每条命令在执行成功之后，都会立即调用 fsync 或 fdatasync 将事务数据写入到 AOF 文件。但是，这种保存是由后台线程进行的，主线程不会阻塞直到保存成功，所以从命令执行成功到数据保存到硬盘之间，还是有一段非常小的间隔，所以这种模式下的事务也是不持久的。
- 其他 AOF 模式也和“总是 SYNC ”模式类似，所以它们都是不持久的。

**2. 命令介绍**
下面介绍命令的使用详情：

|命令|作用|
|---|---|
|MULTI|标记一个事务的开始|
|DISCARD|取消事务，放弃执行事务块内的所有命令|
|EXEC|执行事务内的所有命令|
|WATCH key [key …]|监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断，不会被执行|
|UNWATCH|取消 WATCH 命令对所有 key 的监视|

**3. 实现细节**
Redis事务从开始到结束通常分为三步：

1. 事务开始(MULTI)：MULTI命令可以将执行该命令的客户端从非事务状态切换成事务状态，这一切换是通过客户端状态的flags属性中打开 `CLIENT_MULTI` 标识完成的。
2. 命令入队：切换到事务状态后，该客户端输入的所有命令，都会被暂存到一个命令队列里，不会立即执行。
3. 事务执行(EXEX)：EXEC命令将命令队列里的命令挨个执行完成。

Redis会把每个连接的客户端封装成一个client结构体，该结构体包含大量的字段用来保存需要的信息。其中，事务相关的字段如下：

```c
typedef struct client {
    // ...
    multiState mstate;
    list *watched_keys; // 监视的key列表（节点结构：watchedKey）
    // ...
}

/* Client MULTI/EXEC state */
typedef struct multiCmd {
    robj **argv;    /* 参数 */
    int argc;       /* 参数数量 */
    struct redisCommand *cmd;   /* 命令本身 */
} multiCmd;

typedef struct multiState {
    multiCmd *commands;     /* 事务命令队列 */
    int count;              /* 队列中命令的数量 */
    int cmd_flags;          /*  */
    int minreplicas;        /* 需要同步复制的最小数量 */
    time_t minreplicas_timeout; /* 同步复制超时时间 */
} multiState;

// 监视列表中的节点结构
typedef struct watchedKey {
    robj *key;      /* watch的key*/
    redisDb *db;    /* 指向的db */
} watchedKey;
```

需要注意的是，客户端打开事务操作标识后，只有命令：EXEC、DISCARD、WATCH、MULTI命令会被立即执行，该逻辑在server.c文件中的processCommand方法中：

```c
int processCommand(client *c) {
    // ...
    /* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        // 将命令插入队列
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        // 立即执行
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
    // ...
}

// 将命令插入队列中
void queueMultiCommand(client *c) {
    multiCmd *mc;
    int j;

    c->mstate.commands = zrealloc(c->mstate.commands,
            sizeof(multiCmd)*(c->mstate.count+1));
    mc = c->mstate.commands+c->mstate.count;
    mc->cmd = c->cmd;
    mc->argc = c->argc;
    mc->argv = zmalloc(sizeof(robj*)*c->argc);
    memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    c->mstate.count++;
    c->mstate.cmd_flags |= c->cmd->flags;
}
```

最后我们考虑一下watch机制的触发时机，现在我们已经把想要watch的key加入到了watch的数据结构中，可以想到触发watch的时机应该是修改key的内容时，通知到所有watch了该key的客户端。该触发机制的源码在multi.c文件的`touchWatchedKey()`函数中实现。

</br>
</br>

## 函数功能总览

---

```c
void multiCommand(client *c); // 设置客户端事务标识
void execCommand(client *c); // 执行命令队列中的命令
void discardCommand(client *c); // 取消事务
void watchCommand(client *c); // 监视一个或多个key
void unwatchCommand(client *c); // 取消监视
```

</br>

## 主要函数实现

---

**事务开始：**

```c
void multiCommand(client *c) {
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    // 设置客户端的事务标识
    c->flags |= CLIENT_MULTI;
    addReply(c,shared.ok);
}
```

**执行事务：**

```c
void execCommand(client *c) {
    int j;
    robj **orig_argv;
    int orig_argc;
    struct redisCommand *orig_cmd;
    int must_propagate = 0; // 标记是否需要把MULTI/EXEC传递到AOF或者slaves节点
    int was_master = server.masterhost == NULL; // 标记当前redis节点是否为主节点

    // 如果客户端没有处于事务状态，则返回错误提示信息
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"EXEC without MULTI");
        return;
    }

    /* 首先对两个需要终止当前事务的条件进行判断:
     * 1) 当有WATCH的key被修改时则终止，返回一个nullmultibulk对象
     * 2) 当之前有命令加入事务命令数组出错则终止，例如传入的命令参数数量不对，会返回execaborterr */
    if (c->flags & (CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)) {
        addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
                                                   shared.nullarray[c->resp]);
        // 删除当前事务信息
        discardTransaction(c);
        goto handle_monitor;
    }

    /* 如果事务命令中有写的操作，并且当前redis节点为只读slave节点，将返回错误 */
    if (!server.loading && server.masterhost && server.repl_slave_ro &&
        !(c->flags & CLIENT_MASTER) && c->mstate.cmd_flags & CMD_WRITE)
    {
        addReplyError(c,
            "Transaction contains write commands but instance "
            "is now a read-only replica. EXEC aborted.");
        discardTransaction(c);
        goto handle_monitor;
    }

    /* 执行队列中的所有命令 */
    unwatchAllKeys(c); /* 把watch的key都删除 */
    // 保存当前命令上下文
    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyArrayLen(c,c->mstate.count);
    for (j = 0; j < c->mstate.count; j++) {
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd;

        /* 同步事务操作到AOF或者集群中的从节点. */
        if (!must_propagate && !(c->cmd->flags & (CMD_READONLY|CMD_ADMIN))) {
            execCommandPropagateMulti(c);
            must_propagate = 1;
        }

        int acl_keypos;
        int acl_retval = ACLCheckCommandPerm(c,&acl_keypos);
        if (acl_retval != ACL_OK) {
            addACLLogEntry(c,acl_retval,acl_keypos,NULL);
            addReplyErrorFormat(c,
                "-NOPERM ACLs rules changed between the moment the "
                "transaction was accumulated and the EXEC call. "
                "This command is no longer allowed for the "
                "following reason: %s",
                (acl_retval == ACL_DENIED_CMD) ?
                "no permission to execute the command or subcommand" :
                "no permission to touch the specified keys");
        } else {
            call(c,server.loading ? CMD_CALL_NONE : CMD_CALL_FULL);
        }

        /* 由于命令可以修改参数的值或者数量，因此重新保存命令上下文 */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }
    // 恢复原始命令上下文
    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    // 事务执行完成，删除该事务
    discardTransaction(c);

    /* 确保EXEC会进行传递 */
    if (must_propagate) {
        int is_master = server.masterhost == NULL;
        server.dirty++;
        if (server.repl_backlog && was_master && !is_master) {
            char *execcmd = "*1\r\n$4\r\nEXEC\r\n";
            feedReplicationBacklog(execcmd,strlen(execcmd));
        }
    }

handle_monitor:
    /* monitor命令操作 */
    if (listLength(server.monitors) && !server.loading)
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
}
```

**取消事务：**

```c
void discardCommand(client *c) {
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"DISCARD without MULTI");
        return;
    }
    discardTransaction(c);
    addReply(c,shared.ok);
}
// 具体的删除逻辑
void discardTransaction(client *c) {
    freeClientMultiState(c);
    initClientMultiState(c);
    // 状态位还原
    c->flags &= ~(CLIENT_MULTI|CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC);
    unwatchAllKeys(c);
}
// 释放事务中的所有命令
void freeClientMultiState(client *c) {
    int j;

    for (j = 0; j < c->mstate.count; j++) {
        int i;
        multiCmd *mc = c->mstate.commands+j;

        for (i = 0; i < mc->argc; i++)
            decrRefCount(mc->argv[i]);
        zfree(mc->argv);
    }
    zfree(c->mstate.commands);
}
// 事务相关字段设为初始值
void initClientMultiState(client *c) {
    c->mstate.commands = NULL;
    c->mstate.count = 0;
    c->mstate.cmd_flags = 0;
}
```

**watch监视：**

```c
void watchCommand(client *c) {
    int j;

    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"WATCH inside MULTI is not allowed");
        return;
    }
    for (j = 1; j < c->argc; j++)
        watchForKey(c,c->argv[j]);
    addReply(c,shared.ok);
}

void watchForKey(client *c, robj *key) {
    list *clients = NULL;
    listIter li;
    listNode *ln;
    watchedKey *wk;

    /* 判断key是否已经被watch过 */
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        wk = listNodeValue(ln);
        if (wk->db == c->db && equalStringObjects(key,wk->key))
            return; /* Key already watched */
    }
    /* 如果未被watch，则将key加入到列表中
     * 整个watch操作保存了两套数据结构，一套是在db->watched_keys中的字典结构
     */
    clients = dictFetchValue(c->db->watched_keys,key);
    if (!clients) {
        clients = listCreate();
        dictAdd(c->db->watched_keys,key,clients);
        incrRefCount(key);
    }
    listAddNodeTail(clients,c);
    /* 另一套是在c->watched_keys中的链表结构 */
    wk = zmalloc(sizeof(*wk));
    wk->key = key;
    wk->db = c->db;
    incrRefCount(key);
    listAddNodeTail(c->watched_keys,wk);
}
```

**unwatch取消监视：**

```c
void unwatchCommand(client *c) {
    unwatchAllKeys(c);
    // 修改客户端状态
    c->flags &= (~CLIENT_DIRTY_CAS);
    addReply(c,shared.ok);
}

void unwatchAllKeys(client *c) {
    listIter li;
    listNode *ln;
    // 如果客户端没有watch任何key，则直接返回
    if (listLength(c->watched_keys) == 0) return;
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        list *clients;
        watchedKey *wk;

        /* 遍历取出该客户端watch的key */
        wk = listNodeValue(ln);
        clients = dictFetchValue(wk->db->watched_keys, wk->key);
        serverAssertWithInfo(c,NULL,clients != NULL);
        listDelNode(clients,listSearchKey(clients,c));
        /* Kill the entry at all if this was the only client */
        if (listLength(clients) == 0)
            dictDelete(wk->db->watched_keys, wk->key);
        /* Remove this watched key from the client->watched list */
        listDelNode(c->watched_keys,ln);
        decrRefCount(wk->key);
        zfree(wk);
    }
}
```
