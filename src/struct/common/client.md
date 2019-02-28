## 客户端定义
> 位于 src/server.h 文件中。

```c
/* With multiplexing we need to take per-client state.
 * Clients are taken in a linked list. */
typedef struct client {
    uint64_t id;            /* Client incremental unique ID. （客户端唯一 id）*/
    int fd;                 /* Client socket. （客户端描述符）*/
    redisDb *db;            /* Pointer to currently SELECTed DB. （客户端正在使用的数据库编号）*/
    robj *name;             /* As set by CLIENT SETNAME. （客户端名字）*/
    sds querybuf;           /* Buffer we use to accumulate client queries. （客户端查询缓冲）*/
    sds pending_querybuf;   /* If this is a master, this buffer represents the
                               yet not applied replication stream that we
                               are receiving from the master. */
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size. （查询缓冲峰值）*/
    int argc;               /* Num of arguments of current command. （当前命令的参数数量）*/
    robj **argv;            /* Arguments of current command. （当前命令的参数对象数组）*/
    struct redisCommand *cmd, *lastcmd;  /* Last command executed. （客户端执行的命令和最后一次执行的命令）*/
    int reqtype;            /* Request protocol type: PROTO_REQ_* （请求的类型，单条指令还是多条指令）*/
    int multibulklen;       /* Number of multi bulk arguments left to read. （剩余未读取命令数量）*/
    long bulklen;           /* Length of bulk argument in multi bulk request. （命令的长度）*/
    list *reply;            /* List of reply objects to send to the client. （客户端回复链表）*/
    unsigned long long reply_bytes; /* Tot bytes of objects in reply list. （回复链表的总大小）*/
    size_t sentlen;         /* Amount of bytes already sent in the current
                               buffer or object being sent. （已发送的字节数）*/
    time_t ctime;           /* Client creation time. （客户端创建的时间）*/
    time_t lastinteraction; /* Time of the last interaction, used for timeout （客户端与服务器最后一次交互的时间）*/
    time_t obuf_soft_limit_reached_time;    /* 客户端输出缓存区超过软限制的时间 */
    int flags;              /* Client flags: CLIENT_* macros. （客户端标识 CLIENT_SLAVE，在 server.h 中定义）*/
    int authenticated;      /* When requirepass is non-NULL. （认证状态，配置文件中密码不为空时）*/
    int replstate;          /* Replication state if this is a slave. （复制状态，如果是 slave）*/
    int repl_put_online_on_ack; /* Install slave write handler on ACK. （）*/
    int repldbfd;           /* Replication DB file descriptor. （主服务器传过来 RDB 文件的文件描述符）*/
    off_t repldboff;        /* Replication DB file offset. （主服务传过来 RDB 文件的偏移量）*/
    off_t repldbsize;       /* Replication DB file size. （主服务器传过来的 RDB 文件的大小）*/
    sds replpreamble;       /* Replication DB preamble. */
    long long read_reploff; /* Read replication offset if this is a master. （如果是主服务器，读取复制偏移量）*/
    long long reploff;      /* Applied replication offset if this is a master. （如果是主服务器，应用复制偏移量）*/
    long long repl_ack_off; /* Replication ack offset, if this is a slave. （如果是从服务器，最后一次发送 ACK 的偏移量）*/
    long long repl_ack_time;/* Replication ack time, if this is a slave. （如果是从服务器，最后一次发送 ACK 的时间）*/
    long long psync_initial_offset; /* FULLRESYNC reply offset other slaves
                                       copying this slave output buffer
                                       should use. */
    char replid[CONFIG_RUN_ID_SIZE+1]; /* Master replication ID (if master). （主服务的复制 id）*/
    int slave_listening_port; /* As configured with: SLAVECONF listening-port （从服务器监听端口号）*/
    char slave_ip[NET_IP_STR_LEN]; /* Optionally given by REPLCONF ip-address （从服务器监听 ip）*/
    int slave_capa;         /* Slave capabilities: SLAVE_CAPA_* bitwise OR. */
    multiState mstate;      /* MULTI/EXEC state （事务状态）*/
    int btype;              /* Type of blocking op if CLIENT_BLOCKED. （阻塞类型）*/
    blockingState bpop;     /* blocking state （阻塞状态）*/
    long long woff;         /* Last write global replication offset. （最后全量复制写入的偏移量）*/
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS （watch 监视的 key）*/
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) （客户端订阅的频道，key 是频道名，value 是 null）*/
    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) （客户端订阅的模式）*/
    sds peerid;             /* Cached peer ID. */

    /* Response buffer */
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```