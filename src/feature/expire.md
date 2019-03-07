## 过期策略
> 位于 src/expire.c

### 存储
redis 中的所有键都保存在 redisDb 中。其中 dict 称为键空间，保存了所有的键值对。过期时间则保存在 expires 中，expires 中的 key 是指向键空间的指针，value 是过期键的到期时间，是一个 unix 时间戳。

##### 数据结构
```c
typedef struct redisDb {
    dict *dict;          /* The keyspace for this DB （数据库的键空间，保存着数据库中所有的键值对）*/
    dict *expires;       /* Timeout of keys with a timeout set （每个键的过期时间，key 是指向键空间的指针，value 是过期时间，unix 时间戳）*/
    dict *blocking_keys; /* Keys with clients waiting for data (BLPOP) （处于阻塞状态的键，value 是一个链表，保存了被阻塞的客户端）*/
    dict *ready_keys;    /* Blocked keys that received a PUSH （可以解除阻塞的 key）*/
    dict *watched_keys;  /* WATCHED keys for MULTI/EXEC CAS （watch 命令监视的 key，value 是一个链表，保存着所有用到 key 的客户端）*/
    int id;              /* Database ID （数据库编号，一般是 0 - 16）*/
    long long avg_ttl;   /* Average TTL, just for stats （数据库的平均 ttl，用于统计信息）*/
} redisDb;
```

### 策略
* 主动清除
* 惰性清除
* 定时删除