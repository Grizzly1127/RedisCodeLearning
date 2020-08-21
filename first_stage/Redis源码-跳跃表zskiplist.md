# zskiplist跳跃表

## 简介

---
源码位置：t_zset.c/server.h

跳跃表是zset有序集合的底层实现之一。  

**节点插入：** 跳跃表中，一个节点的插入，分为三个步骤。  
1.根据新增节点的score值，从最高层开始往下对比查找到新增节点每一层（这里的“每一层”是指以当前跳跃表最高层为准，并不是新增节点的层数，因为此时新增节点还没有随机出层数）的前驱层的节点指针，redis使用update[ZSKIPLIST_MAXLEVEL]数组来记录。计算出从header的每一层到新增节点所经过的节点数，使用rank[ZSKIPLIST_MAXLEVEL]数组来记录。  
2.使用redis的随机算法，随机出新增节点的层数，如果高于当前层数，则更新跳跃表的最高层数、update数组对应的指向和rank数组对应的值（高出那部分的层数rank值为0）。  
3.进行跳跃表的插入，就是对新增节点插入位置相邻的节点间，根据update数组进行更改指针指向的操作，根据rank来计算出对应span的值。  

**节点删除：** 和节点插入的原理差不多，也需要用到update数组。根据score找到对应节点后，将节点抽离出来（根据update数组进行指针的重新指向，对应的span值减1），进行删除操作即可。

**节点更新：** 和节点删除的原理差不多，如果跳跃表中没有该数据，则进行插入操作。  

跳跃表的结构如下：  
![zskiplist结构](../img/zskiplist_.png)  

| 函数          | 作用          | 时间复杂度 |
| ---           |---           |---        |
|zslCreate      |创建跳跃表     |O(1)       |
|zslFree        |释放跳跃表     |O(n)        |
|zslInsert      |插入节点       |平均O(logN)，最坏O(N)|
|zslDelete      |删除节点       |平均O(logN)，最坏O(N)|
|zslGetRank     |返回指定分值和成员在跳跃表中的排位|平均O(logN)，最坏O(N)|
|zslGetElementByRank|返回跳跃表在给定排位的节点|平均O(logN)，最坏O(N)|

</br>
</br>

## 结构体与宏定义

---
**server.h:**

``` c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score; // 分值
    struct zskiplistNode *backward; // 后继指针
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前驱指针
        unsigned long span; // 跨度
    } level[]; // 跳跃表的层
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // header为跳跃表的表头节点， tail为表尾节点
    unsigned long length; // 跳跃表的长度
    int level; // 跳跃表当前的层数
} zskiplist;

#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */ // 跳跃表层数，最大为32层，理论上当数据达到2^64时，才会能达到最顶层，所以完全足够
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */ // 用于计算上层被分配到的概率

/* Input flags. */
#define ZADD_NONE 0
#define ZADD_INCR (1<<0)    /* Increment the score instead of setting it. */
#define ZADD_NX (1<<1)      /* Don't touch elements not already existing. */
#define ZADD_XX (1<<2)      /* Only touch elements already existing. */

/* Output flags. */
#define ZADD_NOP (1<<3)     /* Operation not performed because of conditionals.*/
#define ZADD_NAN (1<<4)     /* Only touch elements already existing. */
#define ZADD_ADDED (1<<5)   /* The element was new and was added. */
#define ZADD_UPDATED (1<<6) /* The element already existed, score updated. */

/* Flags only used by the ZADD command but not by zsetAdd() API: */
#define ZADD_CH (1<<16)      /* Return num of elements added or updated. */

/* Struct to hold a inclusive/exclusive range spec by score comparison. */
typedef struct {
    double min, max;
    int minex, maxex; /* are min or max exclusive? */
} zrangespec;

/* Struct to hold an inclusive/exclusive range spec by lexicographic comparison. */
typedef struct {
    sds min, max;     /* May be set to shared.(minstring|maxstring) */
    int minex, maxex; /* are min or max exclusive? */
} zlexrangespec;
```

</br>
</br>

## 函数功能总览

---
**server.h:**  

``` c
zskiplist *zslCreate(void); // 创建跳跃表，分配指定级别数的跳跃表节点
void zslFree(zskiplist *zsl); // 释放跳跃表
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele); // 跳跃表插入新节点
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node); // 删除节点
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range); // 查找指定范围中的第一个节点
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range); // 查找指定范围中的最后一个节点
int zslValueGteMin(double value, zrangespec *spec);
int zslValueLteMax(double value, zrangespec *spec);
void zslFreeLexRange(zlexrangespec *spec); // 释放
int zslParseLexRange(robj *min, robj *max, zlexrangespec *spec);
zskiplistNode *zslFirstInLexRange(zskiplist *zsl, zlexrangespec *range);
zskiplistNode *zslLastInLexRange(zskiplist *zsl, zlexrangespec *range);
int zslLexValueGteMin(sds value, zlexrangespec *spec);
int zslLexValueLteMax(sds value, zlexrangespec *spec);
unsigned long zslGetRank(zskiplist *zsl, double score, sds o); // 返回指定分值和成员在跳跃表中的排位（位置）
```

</br>
</br>

## 主要函数实现

---

``` c
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'. */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x; // update数组用来记录新增节点的每一层的前驱节点指针
    unsigned int rank[ZSKIPLIST_MAXLEVEL]; // rank数组用来记录从header的每一层到新增节点所经过的节点数
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    // 从最高层开始往下遍历
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // 判断新增节点在跳跃表中所处的位置
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    // 随机生成新增节点的层数
    level = zslRandomLevel();
    // 如果随机生成的层数比当前跳跃表的层数高，则初始化高于跳跃表的最高层数的所有层
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    // 创建新增节点
    x = zslCreateNode(level,score,ele);
    // 根据update数组来进行新增节点的插入操作，根据rank数组来计算新增节点的跨度
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    // 如果新增节点的层数低于跳跃表的层数，则高于新增节点的层数的跨度增加1
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 设置新增节点的前置节点
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

``` c
/* Delete an element with matching score/element from the skiplist.
 * The function returns 1 if the node was found and deleted, otherwise
 * 0 is returned.
 *
 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
 * it is not freed (but just unlinked) and *node is set to the node pointer,
 * so that it is possible for the caller to reuse the node (including the
 * referenced SDS string at node->ele). */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    // 从最高层开始遍历，查找所要删除节点在跳跃表中的位置
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        // 删除节点
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```

``` c
/* Update the score of an elmenent inside the sorted set skiplist.
 * Note that the element must exist and must match 'score'.
 * This function does not update the score in the hash table side, the
 * caller should take care of it.
 *
 * Note that this function attempts to just update the node, in case after
 * the score update, the node would be exactly at the same position.
 * Otherwise the skiplist is modified by removing and re-adding a new
 * element, which is more costly.
 *
 * The function returns the updated element skiplist node pointer. */
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    /* We need to seek to element to update to start: this is useful anyway,
     * we'll have to update or remove it. */
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < curscore ||
                    (x->level[i].forward->score == curscore &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    /* Jump to our element: note that this function assumes that the
     * element with the matching score exists. */
    x = x->level[0].forward;
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

    /* If the node, after the score update, would be still exactly
     * at the same position, we can just update the score without
     * actually removing and re-inserting the element in the skiplist. */
    if ((x->backward == NULL || x->backward->score < newscore) &&
        (x->level[0].forward == NULL || x->level[0].forward->score > newscore))
    {
        x->score = newscore;
        return x;
    }

    /* No way to reuse the old node: we need to remove and insert a new
     * one at a different place. */
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    /* We reused the old node x->ele SDS string, free the node now
     * since zslInsert created a new one. */
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}
```

``` c
// 随机层数算法
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

``` c
/* Free a whole skiplist. */
void zslFree(zskiplist *zsl) {
    zskiplistNode *node = zsl->header->level[0].forward, *next;

    zfree(zsl->header);
    while(node) {
        next = node->level[0].forward;
        zslFreeNode(node);
        node = next;
    }
    zfree(zsl);
}
```

``` c
// 根据rank（排位）来查找跳跃表中指定的节点
/* Finds an element by its rank. The rank argument needs to be 1-based. */
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank)
        {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        if (traversed == rank) {
            return x;
        }
    }
    return NULL;
}
```
