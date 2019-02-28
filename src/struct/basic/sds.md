## SDS
> 简单动态字符串（Simple Dynamic String）
> redis 中使用的字符串，基本上都是 SDS，而不是 C 原生的字符串。

#### SDS 的优势
* 可以在 O(1) 的时间复杂度下获取字符串的长度。
* 可以方便的进行字符串追加操作。

#### 常用操作
|函数名|说明|参数|返回|
|--|--|--|--|
| sdsnew | [创建一个 SDS](#创建一个SDS) | char* | sds |

#### 创建一个SDS
主要流程如下
* 调用 [sdsnew](#sdsnew) 函数创建一个新的 sds
    * 计算传入字节数组的大小。
    * 调用 [sdsnewlen](#sdsnewlen) 创建 sds。
        * 调用 [sdsReqType](#sdsReqType) 函数，获取 sds 的类型，类型是根据字符串的长度判断的（这样做主要是为了节约内存）。
        * 调用 [sdsHdrSize](#sdsHdrSize) 函数，返回字符串大小。
        * 给新生成的字节数组最后加上 `\0`。

##### sdsnew
```c
/* Create a new sds string starting from a null terminated C string. */
sds sdsnew(const char *init) {
    // 获取字符串的长度
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    // 调用 sdsnewlen 创建 sds
    return sdsnewlen(init, initlen);
}
```

##### sdsnewlen
```c
/* Create a new sds string with the content specified by the 'init' pointer
 * and 'initlen'.
 * If NULL is used for 'init' the string is initialized with zero bytes.
 *
 * The string is always null-termined (all the sds strings are, always) so
 * even if you create an sds string with:
 *
 * mystring = sdsnewlen("abc",3);
 *
 * You can print the string with printf() as there is an implicit \0 at the
 * end of the string. However the string is binary safe and can contain
 * \0 characters in the middle, as the length is stored in the sds header. */
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    // 获取 sds 类型
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */

    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1);
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

##### sdsReqType
```c
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
#endif
    return SDS_TYPE_64;
}
```

##### sdsHdrSize
```c
static inline int sdsHdrSize(char type) {
    // type 和 SDS_TYPE_MASK（7） 按位与
    switch(type&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5);
        case SDS_TYPE_8:
            return sizeof(struct sdshdr8);
        case SDS_TYPE_16:
            return sizeof(struct sdshdr16);
        case SDS_TYPE_32:
            return sizeof(struct sdshdr32);
        case SDS_TYPE_64:
            return sizeof(struct sdshdr64);
    }
    return 0;
}
```