# list列表对象

## 简介

---
源码位置：t_list.c/server.h

Redis3.2版本之前，list对象底层是由[ziplist](../first_stage/Redis源码-压缩列表ziplist.md)和[linkedlist](../first_stage/Redis源码-双端链表adlist.md)实现的。3.2版本之后，底层是由[quicklist](../first_stage/Redis源码-快速列表quicklist.md)来实现。

|命令|功能|时间复杂度|
|---|---|---|
|LPUSH|从列表的最左边插入一个或多个元素（列表为空则创建）|O(1)|
|LRUSHX|从列表的最左边插入一个或多个元素（列表为空时不做操作）|O(1)|
|RPUSH|从列表的最右边插入一个或多个元素（列表为空则创建）|O(1)|
|RPUSHX|从列表的最右边插入一个或多个元素（列表为空时不做操作）|O(1)|
|LPOP|从列表的最左边弹出一个元素|O(1)|
|RPOP|从列表的最右边弹出一个元素|O(1)|
|BLPOP|弹出指定的多个列表中第一个元素（lpop阻塞版本）|O(1)|
|BRPOP|弹出指定的多个列表中最后一个元素（rpop阻塞版本）|O(1)|
|RPOPLPUSH|弹出列表A的最后一个元素，并将该元素插入到列表B的首位|O(1)|
|BRPOPLPUSH|弹出列表A的最后一个元素，并将该元素插入到列表B的首位（rpoplpush阻塞版本）|O(1)|
|LINDEX|获取索引位置的元素|平均O(N)，头尾O(1)|
|LRANGE|从列表中获取指定位置的元素|O(S+N)，S是距离列表头部的偏移位置，N为指定范围元素数|
|LINSERT|在列表中的另一个元素前或后插入一个元素|平均O(N)，头部O(1)|
|LSET|设置index位置元素的值为value|平均O(N)，头尾O(1)|
|LTRIM|修剪一个已存在的列表的大小|平均O(N)|
|LREM|从列表中移除count个值为value的元素|O(N)|
|LLEN|获得列表的长度|O(1)|

</br>
</br>

## 函数功能总览

---

``` c
void blpopCommand(client *c); // blpop命令
void brpopCommand(client *c); // brpop命令
void brpoplpushCommand(client *c); // brpoplpush命令
void lpushCommand(client *c); // lpush命令
void rpushCommand(client *c); // rpush命令
void lpushxCommand(client *c); // lpushx命令
void rpushxCommand(client *c); // rpushx命令
void linsertCommand(client *c); // linsert命令
void lpopCommand(client *c); // lpop命令
void rpopCommand(client *c); // rpop命令
void rpoplpushCommand(client *c); // rpoplpush命令
void llenCommand(client *c); // llen命令
void lindexCommand(client *c); // lindex命令
void lrangeCommand(client *c); // lrange命令
void ltrimCommand(client *c); // ltrim命令
void lremCommand(client *c); // lrem命令
void lsetCommand(client *c); // lset命令
```

</br>

## Redis命令实现

---

插入命令：

``` c
LPUSH key value [value ...]
RPUSH key value [value ...]
```

代码：

``` c
void lpushCommand(client *c) {
    pushGenericCommand(c,LIST_HEAD); // 列表头部插入
}

void rpushCommand(client *c) {
    pushGenericCommand(c,LIST_TAIL); // 列表尾部插入
}

void pushGenericCommand(client *c, int where) {
    int j, pushed = 0;
    robj *lobj = lookupKeyWrite(c->db,c->argv[1]); // db中查找key

    if (lobj && lobj->type != OBJ_LIST) { // 判断对象类型
        addReply(c,shared.wrongtypeerr);
        return;
    }

    // 命令解析
    for (j = 2; j < c->argc; j++) {
        if (!lobj) {
            lobj = createQuicklistObject(); // 列表不存在，则创建新的列表对象
            quicklistSetOptions(lobj->ptr, server.list_max_ziplist_size,
                                server.list_compress_depth); // 设置列表压缩节点的大小和压缩深度
            dbAdd(c->db,c->argv[1],lobj); // 加入到db中
        }
        listTypePush(lobj,c->argv[j],where); // 调用quicklistPush函数插入元素
        pushed++;
    }
    addReplyLongLong(c, (lobj ? listTypeLength(lobj) : 0)); // 返回列表长度
    if (pushed) {
        char *event = (where == LIST_HEAD) ? "lpush" : "rpush";

        signalModifiedKey(c->db,c->argv[1]); // 修改信号
        notifyKeyspaceEvent(NOTIFY_LIST,event,c->argv[1],c->db->id); // 事件通知
    }
    server.dirty += pushed; // 存储上次保存前所有数据变动的长度
}
```

其他与插入相关的命令不做代码解析了，可自行查看源码。

``` c
LPUSHX key value // 如果db中有该列表，则调用quicklistPush插入元素到列表头部，无列表则不做操作返回
RPUSHX key value // 如果db中有该列表，则调用quicklistPush插入元素到列表尾部，无列表则不做操作返回
LINSERT key BEFORE|AFTER pivot value // 使用quicklist迭代器查找到列表中对应的位置，然后在指定位置前或后插入元素
RPOPLPUSH source destination // 调用quicklistPopCustom从列表source中弹出最后一个元素，并调用quicklistPush插入到destination列表中
BRPOPLPUSH source destination timeout // timeout时间内，如果有source列表，则调用rpoplpushCommand函数，否则返回不做操作返回
LSET key index value // 调用quicklistReplaceAtIndex修改指定位置的值
```

</br>

---

弹出命令：

``` c
LPOP key
RPOP key
```

代码：  

``` c
void lpopCommand(client *c) {
    popGenericCommand(c,LIST_HEAD); // 列表头部弹出
}

void rpopCommand(client *c) {
    popGenericCommand(c,LIST_TAIL); // 列表尾部弹出
}

void popGenericCommand(client *c, int where) {
    robj *o = lookupKeyWriteOrReply(c,c->argv[1],shared.null[c->resp]); // db中查找key
    if (o == NULL || checkType(c,o,OBJ_LIST)) return;

    robj *value = listTypePop(o,where); // 调用quicklistPopCustom弹出指定位置的元素
    if (value == NULL) {
        addReplyNull(c);
    } else {
        char *event = (where == LIST_HEAD) ? "lpop" : "rpop";

        addReplyBulk(c,value);
        decrRefCount(value); // 元素value引用计数自减
        notifyKeyspaceEvent(NOTIFY_LIST,event,c->argv[1],c->db->id); // 事件通知
        if (listTypeLength(o) == 0) {
            notifyKeyspaceEvent(NOTIFY_GENERIC,"del",
                                c->argv[1],c->db->id);
            dbDelete(c->db,c->argv[1]); // 如果列表长度为0，则删除改列表对象
        }
        signalModifiedKey(c->db,c->argv[1]);
        server.dirty++;
    }
}
```

其他与插入相关的命令不做代码解析了，可自行查看源码。

``` c
BLPOP key [key ...] timeout
BRPOP key [key ...] timeout
```

</br>

---

获取元素命令：

``` c
LINDEX key index
LRANGE key start stop
```

代码：  

``` c
// LINDEX
void lindexCommand(client *c) {
    robj *o = lookupKeyReadOrReply(c,c->argv[1],shared.null[c->resp]);
    if (o == NULL || checkType(c,o,OBJ_LIST)) return;
    long index;
    robj *value = NULL;

    if ((getLongFromObjectOrReply(c, c->argv[2], &index, NULL) != C_OK))
        return;

    if (o->encoding == OBJ_ENCODING_QUICKLIST) {
        quicklistEntry entry;
        if (quicklistIndex(o->ptr, index, &entry)) { // 获取列表中的指定index位置的元素
            if (entry.value) {
                value = createStringObject((char*)entry.value,entry.sz); // 创建字符串对象
            } else {
                value = createStringObjectFromLongLong(entry.longval); // 创建longlong对象
            }
            addReplyBulk(c,value);
            decrRefCount(value);
        } else {
            addReplyNull(c);
        }
    } else {
        serverPanic("Unknown list encoding");
    }
}

// LRANGE
void lrangeCommand(client *c) {
    robj *o;
    long start, end, llen, rangelen;

    if ((getLongFromObjectOrReply(c, c->argv[2], &start, NULL) != C_OK) ||
        (getLongFromObjectOrReply(c, c->argv[3], &end, NULL) != C_OK)) return;

    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.emptyarray)) == NULL
         || checkType(c,o,OBJ_LIST)) return;
    llen = listTypeLength(o); // 获取列表长度

    /* 将start和end转为正数的索引 */
    if (start < 0) start = llen+start;
    if (end < 0) end = llen+end;
    if (start < 0) start = 0;

    /* Invariant: start >= 0, so this test will be true when end < 0.
     * The range is empty when start > end or start >= length. */
    if (start > end || start >= llen) {
        addReply(c,shared.emptyarray);
        return;
    }
    if (end >= llen) end = llen-1;
    rangelen = (end-start)+1;

    /* Return the result in form of a multi-bulk reply */
    addReplyArrayLen(c,rangelen);
    if (o->encoding == OBJ_ENCODING_QUICKLIST) {
        // 获取列表的迭代器（从start位置开始）
        listTypeIterator *iter = listTypeInitIterator(o, start, LIST_TAIL);

        while(rangelen--) { // 循环遍历获取数据
            listTypeEntry entry;
            listTypeNext(iter, &entry);
            quicklistEntry *qe = &entry.entry;
            if (qe->value) {
                addReplyBulkCBuffer(c,qe->value,qe->sz);
            } else {
                addReplyBulkLongLong(c,qe->longval);
            }
        }
        listTypeReleaseIterator(iter);
    } else {
        serverPanic("List encoding is not QUICKLIST!");
    }
}

```

</br>

---

其他命令：

``` c
LTRIM key start stop
LREM key count value
/*
* LREM count参数：  
* count > 0: 从头往尾移除值为 value 的元素。
* count < 0: 从尾往头移除值为 value 的元素。
* count = 0: 移除所有值为 value 的元素。 */

LLEN key
```

代码：  

``` c
void ltrimCommand(client *c) {
    robj *o;
    long start, end, llen, ltrim, rtrim;

    if ((getLongFromObjectOrReply(c, c->argv[2], &start, NULL) != C_OK) ||
        (getLongFromObjectOrReply(c, c->argv[3], &end, NULL) != C_OK)) return;

    if ((o = lookupKeyWriteOrReply(c,c->argv[1],shared.ok)) == NULL ||
        checkType(c,o,OBJ_LIST)) return;
    llen = listTypeLength(o);

    // 将start和end转为正数的索引
    if (start < 0) start = llen+start;
    if (end < 0) end = llen+end;
    if (start < 0) start = 0;

    // 计算需要修剪的位置
    if (start > end || start >= llen) {
        /* Out of range start or start > end result in empty list */
        ltrim = llen;
        rtrim = 0;
    } else {
        if (end >= llen) end = llen-1;
        ltrim = start;
        rtrim = llen-end-1;
    }

    // 删除掉列表中需要修剪的范围
    if (o->encoding == OBJ_ENCODING_QUICKLIST) {
        quicklistDelRange(o->ptr,0,ltrim);
        quicklistDelRange(o->ptr,-rtrim,rtrim);
    } else {
        serverPanic("Unknown list encoding");
    }

    notifyKeyspaceEvent(NOTIFY_LIST,"ltrim",c->argv[1],c->db->id);
    if (listTypeLength(o) == 0) {
        dbDelete(c->db,c->argv[1]);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",c->argv[1],c->db->id);
    }
    signalModifiedKey(c->db,c->argv[1]);
    server.dirty++;
    addReply(c,shared.ok);
}

void lremCommand(client *c) {
    robj *subject, *obj;
    obj = c->argv[3];
    long toremove;
    long removed = 0;

    if ((getLongFromObjectOrReply(c, c->argv[2], &toremove, NULL) != C_OK))
        return;

    subject = lookupKeyWriteOrReply(c,c->argv[1],shared.czero);
    if (subject == NULL || checkType(c,subject,OBJ_LIST)) return;

    listTypeIterator *li;
    if (toremove < 0) {
        toremove = -toremove;
        li = listTypeInitIterator(subject,-1,LIST_HEAD);
    } else {
        li = listTypeInitIterator(subject,0,LIST_TAIL);
    }

    listTypeEntry entry;
    while (listTypeNext(li,&entry)) {
        if (listTypeEqual(&entry,obj)) {
            listTypeDelete(li, &entry);
            server.dirty++;
            removed++;
            if (toremove && removed == toremove) break;
        }
    }
    listTypeReleaseIterator(li);

    if (removed) {
        signalModifiedKey(c->db,c->argv[1]);
        notifyKeyspaceEvent(NOTIFY_LIST,"lrem",c->argv[1],c->db->id);
    }

    if (listTypeLength(subject) == 0) {
        dbDelete(c->db,c->argv[1]);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",c->argv[1],c->db->id);
    }

    addReplyLongLong(c,removed);
}

void llenCommand(client *c) {
    robj *o = lookupKeyReadOrReply(c,c->argv[1],shared.czero);
    if (o == NULL || checkType(c,o,OBJ_LIST)) return;
    addReplyLongLong(c,listTypeLength(o));
}
```
