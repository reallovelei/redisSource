## 一次请求的完整周期

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