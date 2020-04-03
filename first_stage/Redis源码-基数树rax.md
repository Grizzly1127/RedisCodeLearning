# rax基数树

## 简介

---
源码位置：rax.c/rax.h

Redis 5.0版本引入的一个新的数据结构。  

</br>
</br>

## 结构体与宏定义

---

``` c
#define RAX_NODE_MAX_SIZE ((1<<29)-1)
typedef struct raxNode {
    uint32_t iskey:1;       // 该节点是否包含key，1个bit
    uint32_t isnull:1;      // 该节点的value是否为null，1个bit
    uint32_t iscompr:1;     // 该节点是否压缩，1个bit
    uint32_t size:29;       // 如果是压缩节点，则保存字符串的长度，否则保存子节点的数量，29个bit
    unsigned char data[];   // 柔性数组，保存节点对应的数据，0bit
} raxNode; // raxNode size：4byte

typedef struct rax {
    raxNode *head;      // 头节点
    uint64_t numele;    // 元素个数
    uint64_t numnodes;  // 节点数
} rax;

#define RAX_STACK_STATIC_ITEMS 32
typedef struct raxStack {
    void **stack;
    size_t items, maxitems;
    void *static_items[RAX_STACK_STATIC_ITEMS];
    int oom;
} raxStack;

#define RAX_ITER_STATIC_LEN 128
#define RAX_ITER_JUST_SEEKED (1<<0)
#define RAX_ITER_EOF (1<<1)
#define RAX_ITER_SAFE (1<<2)
typedef struct raxIterator {
    int flags;
    rax *rt;                // 需要迭代的基数树
    unsigned char *key;     /* The current string. */
    void *data;             /* Data associated to this key. */
    size_t key_len;         /* Current key length. */
    size_t key_max;         /* Max key len the current key buffer can hold. */
    unsigned char key_static_string[RAX_ITER_STATIC_LEN];
    raxNode *node;          /* Current node. Only for unsafe iteration. */
    raxStack stack;         /* Stack used for unsafe iteration. */
    raxNodeCallback node_cb; /* Optional node callback. Normally set to NULL. */
} raxIterator;
```

</br>

## 函数功能总览

``` c
rax *raxNew(void);
int raxInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old);
int raxTryInsert(rax *rax, unsigned char *s, size_t len, void *data, void **old);
int raxRemove(rax *rax, unsigned char *s, size_t len, void **old);
void *raxFind(rax *rax, unsigned char *s, size_t len);
void raxFree(rax *rax);
void raxFreeWithCallback(rax *rax, void (*free_callback)(void*));
void raxStart(raxIterator *it, rax *rt);
int raxSeek(raxIterator *it, const char *op, unsigned char *ele, size_t len);
int raxNext(raxIterator *it);
int raxPrev(raxIterator *it);
int raxRandomWalk(raxIterator *it, size_t steps);
int raxCompare(raxIterator *iter, const char *op, unsigned char *key, size_t key_len);
void raxStop(raxIterator *it);
int raxEOF(raxIterator *it);
void raxShow(rax *rax);
uint64_t raxSize(rax *rax);
unsigned long raxTouch(raxNode *n);
void raxSetDebugMsg(int onoff);

/* Internal API. May be used by the node callback in order to access rax nodes
 * in a low level way, so this function is exported as well. */
void raxSetData(raxNode *n, void *data);
```

</br>

## 主要函数实现

---

``` c
// 为了内存对齐填充的字节，为什么需要填充，本质上来说是为了提高cpu性能。填充之后的数据首地址按照sizeof(void*)字节对齐
#define raxPadding(nodesize) ((sizeof(void*)-((nodesize+4) % sizeof(void*))) & (sizeof(void*)-1))

// 创建基数树
rax *raxNew(void) {
    rax *rax = rax_malloc(sizeof(*rax)); // sizeof(*rax) = 4
    if (rax == NULL) return NULL;
    rax->numele = 0;
    rax->numnodes = 1;
    rax->head = raxNewNode(0,0); // 创建首节点
    if (rax->head == NULL) {
        rax_free(rax);
        return NULL;
    } else {
        return rax;
    }
}
// 创建一个节点
raxNode *raxNewNode(size_t children, int datafield) {
    size_t nodesize = sizeof(raxNode)+children+raxPadding(children)+
                      sizeof(raxNode*)*children;
    if (datafield) nodesize += sizeof(void*);
    raxNode *node = rax_malloc(nodesize);
    if (node == NULL) return NULL;
    node->iskey = 0;
    node->isnull = 0;
    node->iscompr = 0;
    node->size = children;
    return node;
}
```
