## dict(字典)
> 定义位于 src/dict.h 文件中

### 类型定义
redis 中的字典定义，主要涉及下面 4 个数据结构，主要包含关系如下。
* dict（字典定义）
    * dictType（字典类型，主要包含字典特定的函数）
    * dictht（哈希表）
        * dictEntry（哈希表节点）

```c
// 定义哈希表节点
typedef struct dictEntry {
    void *key;          /* key */
    union {             /* value */
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    // 指向下一个节点的指针
    struct dictEntry *next;
} dictEntry;

// 定义操作哈希表函数
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);                              /* 计算哈希值 */
    void *(*keyDup)(void *privdata, const void *key);                       /* 复制键 */
    void *(*valDup)(void *privdata, const void *obj);                       /* 复制 value */
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);  /* 对比键 */
    void (*keyDestructor)(void *privdata, void *key);                       /* 销毁键 */
    void (*valDestructor)(void *privdata, void *obj);                       /* 销毁 value */
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
// 定义哈希表
typedef struct dictht {
    dictEntry **table;          /* 哈希表 */
    unsigned long size;         /* 哈希表的大小 */
    unsigned long sizemask;     /* 掩码 */
    unsigned long used;         /* 已经使用的节点数量 */
} dictht;

// 字典的定义，包含两个 hashTable，用于渐进式 rehash
typedef struct dict {
    dictType *type; /* 字典类型，保存了字典特定函数 */
    void *privdata; /* 私有数据 */
    dictht ht[2];   /* 哈希表 */
    long rehashidx; /* rehashing not in progress if rehashidx == -1 （表示 rehash 的索引，没有 rehash 时为 -1）*/
    unsigned long iterators; /* number of iterators currently running （目前运行的迭代器数量）*/
} dict;
```

### 常用操作
|函数名|说明|传入参数|返回值|
|--|--|--|--|
|dictCreate|[创建一个字典](#创建一个字典)| dictType*，void* |dict|

##### 创建一个字典
```c
/* Create a new hash table */
dict *dictCreate(dictType *type, void *privDataPtr)
{
    // 申请内存空间
    dict *d = zmalloc(sizeof(*d));

    // 字典初始化
    _dictInit(d,type,privDataPtr);
    return d;
}

/* Initialize the hash table */
int _dictInit(dict *d, dictType *type, void *privDataPtr)
{
    // 初始化两个哈希表
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    // 常量 0
    return DICT_OK;
}

/* Reset a hash table already initialized with ht_init().
 * NOTE: This function should only be called by ht_destroy(). */
static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```