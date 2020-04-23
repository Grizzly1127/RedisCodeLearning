# t_list列表对象

## 简介

---
源码位置：t_list.c/server.h

Redis3.2版本之前，list对象底层是由[ziplist](../first_stage/Redis源码-压缩列表ziplist.md)和[linkedlist](../first_stage/Redis源码-双端链表adlist.md)实现的。3.2版本之后，底层是由[quicklist](../first_stage/Redis源码-快速列表quicklist.md)来实现。

</br>
</br>

## 函数功能总览

---

``` c
void blpopCommand(client *c); // blpop命令，删除并获得指定的多个列表中第一个元素（lpop阻塞版本）
void brpopCommand(client *c); // brpop命令，删除并获得指定的多个列表中最后一个元素（rpop阻塞版本）
void brpoplpushCommand(client *c); // brpoplpush命令，弹出一个列表的值，将它推到另一个列表，并返回它（rpoplpush阻塞版本）
void lpushCommand(client *c); // lpush命令，从列表的最左边插入一个或多个元素（列表为空则创建）
void rpushCommand(client *c); // rpush命令，从列表的最右边插入一个或多个元素（列表为空则创建）
void lpushxCommand(client *c); // lpushx命令，从列表的最左边插入一个或多个元素（列表为空时不做操作）
void rpushxCommand(client *c); // rpushx命令，从列表的最右边插入一个或多个元素（列表为空时不做操作）
void linsertCommand(client *c); // linsert命令，在列表中的另一个元素前或后插入一个元素
void lpopCommand(client *c); // lpop命令，从列表的最左边弹出一个元素
void rpopCommand(client *c); // rpop命令，从列表的最右边弹出一个元素
void rpoplpushCommand(client *c); // rpoplpush命令，移除列表A的最后一个元素，并将该元素插入到列表B的首位
void llenCommand(client *c); // llen命令，获得列表的长度
void lindexCommand(client *c); // lindex命令，获取索引位置的元素
void lrangeCommand(client *c); // lrange命令，从列表中获取指定位置的元素
void ltrimCommand(client *c); // ltrim命令，修剪一个已存在的列表
void lremCommand(client *c); // lrem命令，从列表中移除count个值为value的元素
void lsetCommand(client *c); // lset命令，设置index位置元素的值为value
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
