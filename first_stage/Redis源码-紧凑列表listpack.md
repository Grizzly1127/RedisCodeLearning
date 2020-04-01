# listpack紧凑列表

## 简介

---
源码位置：listpack.c/listpack.h

Redis 5.0版本引入的一个新的数据结构

</br>
</br>

## 结构体与宏定义

``` c
// listpack_malloc.h
#define lp_malloc zmalloc
#define lp_realloc zrealloc
#define lp_free zfree

#define LP_INTBUF_SIZE 21 /* 20 digits of -2^63 + 1 null term = 21. */

// listpack.h
/* lpInsert() where argument possible values: */
#define LP_BEFORE 0
#define LP_AFTER 1
#define LP_REPLACE 2

// listpack.c
#define LP_HDR_SIZE 6       /* 32 bit total len + 16 bit number of elements. */ // 前4个byte表示占用的总字节数，后2个byte表示元素个数
#define LP_HDR_NUMELE_UNKNOWN UINT16_MAX
#define LP_MAX_INT_ENCODING_LEN 9
#define LP_MAX_BACKLEN_SIZE 5
#define LP_MAX_ENTRY_BACKLEN 34359738367ULL
#define LP_ENCODING_INT 0
#define LP_ENCODING_STRING 1

#define LP_ENCODING_7BIT_UINT 0
#define LP_ENCODING_7BIT_UINT_MASK 0x80 // 1000 0000
#define LP_ENCODING_IS_7BIT_UINT(byte) (((byte)&LP_ENCODING_7BIT_UINT_MASK)==LP_ENCODING_7BIT_UINT)

#define LP_ENCODING_6BIT_STR 0x80 // 1000 0000
#define LP_ENCODING_6BIT_STR_MASK 0xC0 // 1100 0000
#define LP_ENCODING_IS_6BIT_STR(byte) (((byte)&LP_ENCODING_6BIT_STR_MASK)==LP_ENCODING_6BIT_STR)

#define LP_ENCODING_13BIT_INT 0xC0 // 1100 0000
#define LP_ENCODING_13BIT_INT_MASK 0xE0 // 1110 0000
#define LP_ENCODING_IS_13BIT_INT(byte) (((byte)&LP_ENCODING_13BIT_INT_MASK)==LP_ENCODING_13BIT_INT)

#define LP_ENCODING_12BIT_STR 0xE0 // 1110 0000
#define LP_ENCODING_12BIT_STR_MASK 0xF0 // 1111 0000
#define LP_ENCODING_IS_12BIT_STR(byte) (((byte)&LP_ENCODING_12BIT_STR_MASK)==LP_ENCODING_12BIT_STR)

#define LP_ENCODING_16BIT_INT 0xF1 // 1111 0001
#define LP_ENCODING_16BIT_INT_MASK 0xFF // 1111 1111
#define LP_ENCODING_IS_16BIT_INT(byte) (((byte)&LP_ENCODING_16BIT_INT_MASK)==LP_ENCODING_16BIT_INT)

#define LP_ENCODING_24BIT_INT 0xF2 // 1111 0010
#define LP_ENCODING_24BIT_INT_MASK 0xFF // 1111 1111
#define LP_ENCODING_IS_24BIT_INT(byte) (((byte)&LP_ENCODING_24BIT_INT_MASK)==LP_ENCODING_24BIT_INT)

#define LP_ENCODING_32BIT_INT 0xF3 // 1111 0011
#define LP_ENCODING_32BIT_INT_MASK 0xFF // 1111 1111
#define LP_ENCODING_IS_32BIT_INT(byte) (((byte)&LP_ENCODING_32BIT_INT_MASK)==LP_ENCODING_32BIT_INT)

#define LP_ENCODING_64BIT_INT 0xF4 // 1111 0100
#define LP_ENCODING_64BIT_INT_MASK 0xFF // 1111 1111
#define LP_ENCODING_IS_64BIT_INT(byte) (((byte)&LP_ENCODING_64BIT_INT_MASK)==LP_ENCODING_64BIT_INT)

#define LP_ENCODING_32BIT_STR 0xF0 // 1111 0000
#define LP_ENCODING_32BIT_STR_MASK 0xFF  // 1111 1111
#define LP_ENCODING_IS_32BIT_STR(byte) (((byte)&LP_ENCODING_32BIT_STR_MASK)==LP_ENCODING_32BIT_STR)

#define LP_EOF 0xFF // listpack结束标志

#define LP_ENCODING_6BIT_STR_LEN(p) ((p)[0] & 0x3F)
#define LP_ENCODING_12BIT_STR_LEN(p) ((((p)[0] & 0xF) << 8) | (p)[1])
#define LP_ENCODING_32BIT_STR_LEN(p) (((uint32_t)(p)[1]<<0) | \
                                      ((uint32_t)(p)[2]<<8) | \
                                      ((uint32_t)(p)[3]<<16) | \
                                      ((uint32_t)(p)[4]<<24))
// 获取listpack总字节数
#define lpGetTotalBytes(p)           (((uint32_t)(p)[0]<<0) | \
                                      ((uint32_t)(p)[1]<<8) | \
                                      ((uint32_t)(p)[2]<<16) | \
                                      ((uint32_t)(p)[3]<<24))
// 获取listpack元素个数
#define lpGetNumElements(p)          (((uint32_t)(p)[4]<<0) | \
                                      ((uint32_t)(p)[5]<<8))
// 设置listpack的总字节数
#define lpSetTotalBytes(p,v) do { \
    (p)[0] = (v)&0xff; \
    (p)[1] = ((v)>>8)&0xff; \
    (p)[2] = ((v)>>16)&0xff; \
    (p)[3] = ((v)>>24)&0xff; \
} while(0)
// 设置元素个数
#define lpSetNumElements(p,v) do { \
    (p)[4] = (v)&0xff; \
    (p)[5] = ((v)>>8)&0xff; \
} while(0)
```

</br>

## 函数功能总览

``` c
unsigned char *lpNew(void);
void lpFree(unsigned char *lp);
unsigned char *lpInsert(unsigned char *lp, unsigned char *ele, uint32_t size, unsigned char *p, int where, unsigned char **newp);
unsigned char *lpAppend(unsigned char *lp, unsigned char *ele, uint32_t size);
unsigned char *lpDelete(unsigned char *lp, unsigned char *p, unsigned char **newp);
uint32_t lpLength(unsigned char *lp);
unsigned char *lpGet(unsigned char *p, int64_t *count, unsigned char *intbuf);
unsigned char *lpFirst(unsigned char *lp);
unsigned char *lpLast(unsigned char *lp);
unsigned char *lpNext(unsigned char *lp, unsigned char *p);
unsigned char *lpPrev(unsigned char *lp, unsigned char *p);
uint32_t lpBytes(unsigned char *lp);
unsigned char *lpSeek(unsigned char *lp, long index);
```

</br>

## 主要函数实现

---

``` c
// 创建listpack
unsigned char *lpNew(void) {
    unsigned char *lp = lp_malloc(LP_HDR_SIZE+1); // 申请7byte的内存空间
    if (lp == NULL) return NULL;
    lpSetTotalBytes(lp,LP_HDR_SIZE+1);
    lpSetNumElements(lp,0);
    lp[LP_HDR_SIZE] = LP_EOF; // 节点结束标志（恒为0xFF）
    return lp;
}

// 释放listpack
void lpFree(unsigned char *lp) {
    lp_free(lp);
}
```

**编码相关：**

``` c
// 获取元素编码类型和长度
int lpEncodeGetType(unsigned char *ele, uint32_t size, unsigned char *intenc, uint64_t *enclen) {
    int64_t v;
    if (lpStringToInt64((const char*)ele, size, &v)) {
        // 如果能转为整数，则判定元素为整数，根据数值编码到intenc中
        if (v >= 0 && v <= 127) {
            /* Single byte 0-127 integer. */
            intenc[0] = v;
            *enclen = 1;
        } else if (v >= -4096 && v <= 4095) {
            /* 13 bit integer. */
            if (v < 0) v = ((int64_t)1<<13)+v;
            intenc[0] = (v>>8)|LP_ENCODING_13BIT_INT;
            intenc[1] = v&0xff;
            *enclen = 2;
        } else if (v >= -32768 && v <= 32767) {
            /* 16 bit integer. */
            if (v < 0) v = ((int64_t)1<<16)+v;
            intenc[0] = LP_ENCODING_16BIT_INT;
            intenc[1] = v&0xff;
            intenc[2] = v>>8;
            *enclen = 3;
        } else if (v >= -8388608 && v <= 8388607) {
            /* 24 bit integer. */
            if (v < 0) v = ((int64_t)1<<24)+v;
            intenc[0] = LP_ENCODING_24BIT_INT;
            intenc[1] = v&0xff;
            intenc[2] = (v>>8)&0xff;
            intenc[3] = v>>16;
            *enclen = 4;
        } else if (v >= -2147483648 && v <= 2147483647) {
            /* 32 bit integer. */
            if (v < 0) v = ((int64_t)1<<32)+v;
            intenc[0] = LP_ENCODING_32BIT_INT;
            intenc[1] = v&0xff;
            intenc[2] = (v>>8)&0xff;
            intenc[3] = (v>>16)&0xff;
            intenc[4] = v>>24;
            *enclen = 5;
        } else {
            /* 64 bit integer. */
            uint64_t uv = v;
            intenc[0] = LP_ENCODING_64BIT_INT;
            intenc[1] = uv&0xff;
            intenc[2] = (uv>>8)&0xff;
            intenc[3] = (uv>>16)&0xff;
            intenc[4] = (uv>>24)&0xff;
            intenc[5] = (uv>>32)&0xff;
            intenc[6] = (uv>>40)&0xff;
            intenc[7] = (uv>>48)&0xff;
            intenc[8] = uv>>56;
            *enclen = 9;
        }
        return LP_ENCODING_INT;
    } else {
        // 如果是字符串，则计算字符串的编码长度
        if (size < 64) *enclen = 1+size;
        else if (size < 4096) *enclen = 2+size;
        else *enclen = 5+size;
        return LP_ENCODING_STRING;
    }
}

unsigned long lpEncodeBacklen(unsigned char *buf, uint64_t l) {
    if (l <= 127) {
        if (buf) buf[0] = l;
        return 1;
    } else if (l < 16383) {
        if (buf) {
            buf[0] = l>>7;
            buf[1] = (l&127)|128;
        }
        return 2;
    } else if (l < 2097151) {
        if (buf) {
            buf[0] = l>>14;
            buf[1] = ((l>>7)&127)|128;
            buf[2] = (l&127)|128;
        }
        return 3;
    } else if (l < 268435455) {
        if (buf) {
            buf[0] = l>>21;
            buf[1] = ((l>>14)&127)|128;
            buf[2] = ((l>>7)&127)|128;
            buf[3] = (l&127)|128;
        }
        return 4;
    } else {
        if (buf) {
            buf[0] = l>>28;
            buf[1] = ((l>>21)&127)|128;
            buf[2] = ((l>>14)&127)|128;
            buf[3] = ((l>>7)&127)|128;
            buf[4] = (l&127)|128;
        }
        return 5;
    }
}

void lpEncodeString(unsigned char *buf, unsigned char *s, uint32_t len) {
    if (len < 64) {
        buf[0] = len | LP_ENCODING_6BIT_STR;
        memcpy(buf+1,s,len);
    } else if (len < 4096) {
        buf[0] = (len >> 8) | LP_ENCODING_12BIT_STR;
        buf[1] = len & 0xff;
        memcpy(buf+2,s,len);
    } else {
        buf[0] = LP_ENCODING_32BIT_STR;
        buf[1] = len & 0xff;
        buf[2] = (len >> 8) & 0xff;
        buf[3] = (len >> 16) & 0xff;
        buf[4] = (len >> 24) & 0xff;
        memcpy(buf+5,s,len);
    }
}
```

``` c
// 插入元素
unsigned char *lpInsert(unsigned char *lp, unsigned char *ele, uint32_t size, unsigned char *p, int where, unsigned char **newp) {
    unsigned char intenc[LP_MAX_INT_ENCODING_LEN];
    unsigned char backlen[LP_MAX_BACKLEN_SIZE];

    uint64_t enclen; /* The length of the encoded element. */

    /* An element pointer set to NULL means deletion, which is conceptually
     * replacing the element with a zero-length element. So whatever we
     * get passed as 'where', set it to LP_REPLACE. */
    // 如果元素ele为空意味着删除，即用0长度元素去替换，所以where设置为LP_REPLACE
    if (ele == NULL) where = LP_REPLACE;

    /* If we need to insert after the current element, we just jump to the
     * next element (that could be the EOF one) and handle the case of
     * inserting before. So the function will actually deal with just two
     * cases: LP_BEFORE and LP_REPLACE. */
    // 如果想要在位置p后插入元素，则将p跳到下一个元素并在之前处理插入的情况，所以该函数实际上只处理两种情况，LP_BEFORE和LP_REPLACE
    if (where == LP_AFTER) {
        p = lpSkip(p);
        where = LP_BEFORE;
    }

    /* Store the offset of the element 'p', so that we can obtain its
     * address again after a reallocation. */
    unsigned long poff = p-lp;

    /* Calling lpEncodeGetType() results into the encoded version of the
     * element to be stored into 'intenc' in case it is representable as
     * an integer: in that case, the function returns LP_ENCODING_INT.
     * Otherwise if LP_ENCODING_STR is returned, we'll have to call
     * lpEncodeString() to actually write the encoded string on place later.
     *
     * Whatever the returned encoding is, 'enclen' is populated with the
     * length of the encoded element. */
    int enctype;
    if (ele) {
        // 获取编码类型和编码长度
        enctype = lpEncodeGetType(ele,size,intenc,&enclen);
    } else {
        enctype = -1;
        enclen = 0;
    }

    /* We need to also encode the backward-parsable length of the element
     * and append it to the end: this allows to traverse the listpack from
     * the end to the start. */
    unsigned long backlen_size = ele ? lpEncodeBacklen(backlen,enclen) : 0; // 编码长度
    uint64_t old_listpack_bytes = lpGetTotalBytes(lp); // 获取当前lp的字节数
    uint32_t replaced_len  = 0;
    if (where == LP_REPLACE) {
        // 如果是删除操作
        replaced_len = lpCurrentEncodedSize(p); // 当前位置p的元素编码字节数
        replaced_len += lpEncodeBacklen(NULL,replaced_len);
    }

    uint64_t new_listpack_bytes = old_listpack_bytes + enclen + backlen_size
                                  - replaced_len;
    if (new_listpack_bytes > UINT32_MAX) return NULL; // 单个listpack最大只能是4GB

    /* We now need to reallocate in order to make space or shrink the
     * allocation (in case 'when' value is LP_REPLACE and the new element is
     * smaller). However we do that before memmoving the memory to
     * make room for the new element if the final allocation will get
     * larger, or we do it after if the final allocation will get smaller. */

    unsigned char *dst = lp + poff; /* May be updated after reallocation. */ // 指向元素将要插入的位置

    /* Realloc before: we need more room. */
    if (new_listpack_bytes > old_listpack_bytes) {
        if ((lp = lp_realloc(lp,new_listpack_bytes)) == NULL) return NULL;
        dst = lp + poff;
    }

    /* Setup the listpack relocating the elements to make the exact room
     * we need to store the new one. */
    if (where == LP_BEFORE) {
        memmove(dst+enclen+backlen_size,dst,old_listpack_bytes-poff);
    } else { /* LP_REPLACE. */
        long lendiff = (enclen+backlen_size)-replaced_len;
        memmove(dst+replaced_len+lendiff,
                dst+replaced_len,
                old_listpack_bytes-poff-replaced_len);
    }

    /* Realloc after: we need to free space. */
    if (new_listpack_bytes < old_listpack_bytes) {
        if ((lp = lp_realloc(lp,new_listpack_bytes)) == NULL) return NULL;
        dst = lp + poff;
    }

    /* Store the entry. */
    if (newp) {
        *newp = dst;
        /* In case of deletion, set 'newp' to NULL if the next element is
         * the EOF element. */
        if (!ele && dst[0] == LP_EOF) *newp = NULL;
    }
    if (ele) {
        if (enctype == LP_ENCODING_INT) {
            memcpy(dst,intenc,enclen);
        } else {
            lpEncodeString(dst,ele,size);
        }
        dst += enclen;
        memcpy(dst,backlen,backlen_size);
        dst += backlen_size;
    }

    /* Update header. */
    if (where != LP_REPLACE || ele == NULL) {
        uint32_t num_elements = lpGetNumElements(lp);
        if (num_elements != LP_HDR_NUMELE_UNKNOWN) {
            if (ele)
                lpSetNumElements(lp,num_elements+1);
            else
                lpSetNumElements(lp,num_elements-1);
        }
    }
    lpSetTotalBytes(lp,new_listpack_bytes);

#if 0
    /* This code path is normally disabled: what it does is to force listpack
     * to return *always* a new pointer after performing some modification to
     * the listpack, even if the previous allocation was enough. This is useful
     * in order to spot bugs in code using listpacks: by doing so we can find
     * if the caller forgets to set the new pointer where the listpack reference
     * is stored, after an update. */
    unsigned char *oldlp = lp;
    lp = lp_malloc(new_listpack_bytes);
    memcpy(lp,oldlp,new_listpack_bytes);
    if (newp) {
        unsigned long offset = (*newp)-oldlp;
        *newp = lp + offset;
    }
    /* Make sure the old allocation contains garbage. */
    memset(oldlp,'A',new_listpack_bytes);
    lp_free(oldlp);
#endif

    return lp;
}

unsigned char *lpSkip(unsigned char *p) {
    unsigned long entrylen = lpCurrentEncodedSize(p);
    entrylen += lpEncodeBacklen(NULL,entrylen);
    p += entrylen;
    return p;
}
```
