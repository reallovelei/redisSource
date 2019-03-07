## 主从复制
> 文件位于 src/replication.c

redis 的主从复制是由时间事件 serverCron 触发的，在 serverCron 函数中，调用了 [replicationCron](../func/replication/replicationCron.md) 函数，执行主从复制操作，replicationCron 函数每秒会被执行一次。

#### 执行流程
* 如果正在与 master 建立连接，或者连接到 master 超时，则取消复制。
* 如果从 master 接收文件超时，取消复制。
* 与 master 丢失连接，取消复制。
* slave 当前状态是 REPL_STATE_CONNECT（需要与 master 建立连接），则建立连接
* 定期向 master 发送 ACK。
* 如果 master 有 slave，定期向它们发送 PING。
    * 首先，向 slave 发送 PING。
    * 其次，向所有等待 master 创建 RDB 文件的 slave 发送一个空行。
* 断开超时的 slave。
* 如果 master 没有任何 slave，则在一定时间内释放掉。
* 如果关闭了 AOF 且没有任何 slave 连接，释放掉复制脚本的缓存。
* 调用 [startBgsaveForReplication](../func/replication/startBgsaveForReplication.md) 开始 RDB。

#### 常量定义
> 位于 src/server.h
##### 复制相关
```c
/* Slave replication state. Used in server.repl_state for slaves to remember
 * what to do next. */
#define REPL_STATE_NONE 0 /* No active replication */
#define REPL_STATE_CONNECT 1 /* Must connect to master （必须要连接到 master）*/
#define REPL_STATE_CONNECTING 2 /* Connecting to master （正在与 master 建立连接）*/
```

##### SLAVE 相关
```c
#define SLAVE_STATE_WAIT_BGSAVE_START 6 /* We need to produce a new RDB file. （需要生成 RDB 文件）*/
#define SLAVE_STATE_WAIT_BGSAVE_END 7 /* Waiting RDB file creation to finish. （等待 RDB 文件完成）*/
#define SLAVE_STATE_SEND_BULK 8 /* Sending RDB file to slave. （向 slave 发送 RDB 文件）*/
#define SLAVE_STATE_ONLINE 9 /* RDB file transmitted, sending just updates. （RDB 文件已发送）*/
```