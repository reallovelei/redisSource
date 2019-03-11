## 一次请求的完整周期

* [服务端](#服务端)
* [客户端](#客户端)

### 服务端
> 主要文件位于 src/networking 中
主要可以分为这么几个阶段
* [创建客户端](#创建客户端)
* [为新创建的客户端绑定命令处理函数](#绑定命令处理函数)
* [处理命令](#处理命令)
* [执行命令](#执行命令)

#### 创建客户端
在[创建连接](./connect.md)的时候，我们注意到，服务器调用 `createClient` 函数创建了一个客户端。
现在我们来看一下 `createClient` 函数都做了哪些工作，代码可以看[这里](../func/networking/createClient.md)。

* 为新创建的客户端申请内存。
* 根据传入的 fd 判断，创建正常客户端还是伪客户端。
* 为正常客户端绑定 readQueryFromClient 文件事件，用于接收来自客户端的命令请求。
* 初始化客户端配置。

#### 绑定命令处理函数
可以看到，用于接收来自客户端的命令请求，是在 `readQueryFromClient` 函数中处理的。
[readQueryFromClient](../func/networking/readQueryFromClient.md) 的目的是从客户端的查询缓冲中读取数据，概括一下执行的流程。
* 从套接字中读取内容到缓冲区。
* 更新一下客户端与服务器的最后交互时间。
* 调用 [processInputBuffer](../func/networking/processInputBuffer.md) 函数，处理请求。
* 判断客户端是主还是从，如果是从，要接收来自主服务器的复制。

#### 处理命令
* 判断客户端的查询类型，多条查询还是内连查询（分别会调用 `processInlineBuffer` 和 `processMultibulkBuffer` 处理对应的请求）。
* 调用 [processCommand](../func/server/processCommand.md) 函数执行命令。

#### 执行命令
流程如下：
* 对 quit 命令特殊处理
* 检验 key 以及参数是否合法（会在 server.command 字典中查找对应的命令，rename 之后的）。
* 如果设置了密码，验证密码是否合法。
* 如果是集群模式，则特殊处理
    * 调用 `getNodeByQuery` 函数，获得集群的节点，这里会检验命令的所有 key 是否在同一个 slot，如果不是，则会返回 `CLUSTER_REDIR_CROSS_SLOT` 的错误。
    * 如果 key 不属于本节点，返回一条 MOVE 指令。
* 如果设置了最大内存，检验内存是否超出限制。
* 如果当前是主服务器，且 bgsave 发生错误，不允许写入。
* 如果设置了 `min-slaves-to-write` 且没有足够多的从服务器，不允许写入。
* 对应 read-only 的从服务器，不允许写入。
* 发布订阅模式只允许 SUBSCRIBE 和 UNSUBSCRIBE 命令。
* `slave-serve-stale-data`（slave 丢失连接时是否继续处理请求）如果设置为 no，则只允许 info 和 slaveof 命令。
* 正在载入数据，只允许 CMD_LOADING 的命令。
* Lua 脚本太慢？只允许特定的 lua 命令（原文：Lua script too slow? Only allow a limited number of commands）。
* 如果客户端开启了事务，则除 EXEC、DISCARD、MULTI 和 WATCH 命令之外，都放入事务队列 `queueMultiCommand`（位于 src/networking.c））。
    * 之后调用 `addReply` 函数向客户端回复。
    * `addReply` 函数中会调用 `prepareClientToWrite` 函数把客户端的写请求放入 `server.clients_pending_write` 链表。
* 如果未开启事务，则调用 [call](../func/server/call.md) 函数执行命令（事务模式下的 EXEC、DISCARD、MULTI 和 WATCH 也会走到这一步）。
    * call 函数是 redis 执行命令的核心函数，后面会分析一下。

#### 总结
* rename 危险的命令（如 flushdb 等，就是在这一步验证的）。
* 设置了密码，也是在这一步验证的。
* 集群模式下，如果多个 key 不在同一 slot，会返回 CLUSTER_REDIR_CROSS_SLOT 的错误。
* 事务模式下，除 EXEC 、 DISCARD 、 MULTI 和 WATCH 命令，都是调用 `queueMultiCommand` 放入事务队列执行的。
* 非事务模式，或者 EXEC 、 DISCARD 、 MULTI 和 WATCH，调用 `call` 函数执行命令。

上面说到，非事务模式，或者 EXEC 、 DISCARD 、 MULTI 和 WATCH 这四个命令，会调用 call 函数执行。
看一下 call 函数的主要流程

* 如果命令不是来自读取 AOF 文件，发送命令到 monitor 模式下的客户端。
* 初始化。
* 调用 `c->cmd->proc(c)` 执行命令（processCommand 中 `c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);` 会过滤掉非法指令）。
    * 在服务器初始化时，调用了一个 `populateCommandTable` 函数，将 redis 到命令和对应的处理函数关联到一起，这里的 proc 就是对应的处理函数（可以看一下 redisCommandTable 数组的定义）。
* 设置 dirty 标记，用于事务。
* 针对 Lua 调用者做特殊处理。
* 将命令复制到 AOF 和 slave 节点。
* 重置客户端的 flag 标记，因为 call 可能递归调用。
* 更新命令的统计信息。

总结一下
    * 在服务器初始化阶段，会调用 `populateCommandTable` 函数，将 redis 命令和对应的处理函数关联，定义位于 src/server.c 中的 redisCommandTable 数组。
    * 真正处理命令的是 redisCommandTable 数组中定义的函数。
    

### 客户端
> 文件位于 src/redis-cli.c 中
#### 执行流程
> 客户端在和服务器建立连接的时候，最后调用了 `noninteractive` 这个函数。

我们来看一下这个函数的定义。

* 这里判断了一下是否有标准输入，最后都调用了 `issueCommand` 这个函数。

```c
static int noninteractive(int argc, char **argv) {
    int retval = 0;
    // 判断是否有标准输入
    if (config.stdinarg) {
        argv = zrealloc(argv, (argc+1)*sizeof(char*));
        argv[argc] = readArgFromStdin();
        // 发送一个命令
        retval = issueCommand(argc+1, argv);
    } else {
        retval = issueCommand(argc, argv);
    }
    return retval;
}
```

再看一下 `issueCommand` 函数的定义

* `issueCommand` 中，直接调用了 `issueCommandRepeat` 函数。
* `issueCommandRepeat` 函数中，调用了 `cliSendCommand` 函数发送命令，如果失败，会有一次重试。
* `cliSendCommand` 函数中，主要做了这几件事。
    * 对一下特殊指令做一些处理。
    * 如果是 eval 模式，开启对应的选项。
    * 调用 `redisAppendCommandArgv` 函数将参数追加到命令中，并按照 redis 的通信协议格式化命令。
    * 对 pub/sub 模式 和 主从模式做特殊处理。
    * 调用 `cliReadReply` 函数得到回复。
        * 调用 `redisGetReply` 函数（位于 deps/hiredis/hiredis.c 中）
            * 调用 `redisGetReplyFromReader` 从 reader 中读取数据。
            * 调用 `redisBufferWrite` 向 buffer 中写入数据。
            * 调用 `redisBufferRead` 读取数据，当有读事件触发时，这个函数会从 socket 中不断读取数据。
        * 如果是集群模式，并且需要变更槽位，则更新本地槽位映射表，并返回一条 MOVE 指令（这里会将 `config.cluster_reissue_command` 置为 1）。
        * 输出返回结果到标准输出。

```c
static int issueCommand(int argc, char **argv) {
    return issueCommandRepeat(argc, argv, config.repeat);
}

static int issueCommandRepeat(int argc, char **argv, long repeat) {
    while (1) {
        // 集群模式下是否需要向新的节点发送请求
        config.cluster_reissue_command = 0;
        if (cliSendCommand(argc,argv,repeat) != REDIS_OK) {
            cliConnect(1);

            /* If we still cannot send the command print error.
             * We'll try to reconnect the next time. */
            if (cliSendCommand(argc,argv,repeat) != REDIS_OK) {
                cliPrintContextError();
                return REDIS_ERR;
            }
         }
         /* Issue the command again if we got redirected in cluster mode */
         if (config.cluster_mode && config.cluster_reissue_command) {
            cliConnect(1);
         } else {
             break;
        }
    }
    return REDIS_OK;
}
```
