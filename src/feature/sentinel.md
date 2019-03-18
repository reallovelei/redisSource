# sentinel
> 位于 src/sentinel.c 中

* [启动](#启动)
* [执行](#执行)

## 启动
> 我们在服务器的初始化阶段 server.c 中的 main 函数，可以找到下面这样一段代码。
```c
if (server.sentinel_mode) {
    initSentinelConfig();
    initSentinel();
}
```

我们分别来看一下这两个函数到底都做了什么事。
initSentinelConfig 这个函数非常简单，只有一句话，就是初始化了一下 sentinel 的端口号。
```c
/* This function overwrites a few normal Redis config default with Sentinel
 * specific defaults. */
void initSentinelConfig(void) {
    server.port = REDIS_SENTINEL_PORT;
}
```

再来看一下 initSentinel 这个函数，这个函数也不复杂，就直接贴代码了，做的事情也就是初始化一下 sentinel 的一些信息，初始化 sentinel 的命令什么的。
```c
/* Perform the Sentinel mode initialization. */
void initSentinel(void) {
    unsigned int j;

    /* Remove usual Redis commands from the command table, then just add
     * the SENTINEL command. */
    dictEmpty(server.commands,NULL);
    for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {
        int retval;
        struct redisCommand *cmd = sentinelcmds+j;

        retval = dictAdd(server.commands, sdsnew(cmd->name), cmd);
        serverAssert(retval == DICT_OK);
    }

    /* Initialize various data structures. */
    sentinel.current_epoch = 0;
    sentinel.masters = dictCreate(&instancesDictType,NULL);
    sentinel.tilt = 0;
    sentinel.tilt_start_time = 0;
    sentinel.previous_time = mstime();
    sentinel.running_scripts = 0;
    sentinel.scripts_queue = listCreate();
    sentinel.announce_ip = NULL;
    sentinel.announce_port = 0;
    sentinel.simfailure_flags = SENTINEL_SIMFAILURE_NONE;
    memset(sentinel.myid,0,sizeof(sentinel.myid));
}
```

## 执行
> 在服务器的定时任务，serverCron 函数中，可以找到下面这样一段代码。
```c
/* Show information about connected clients */
if (!server.sentinel_mode) {
    run_with_period(5000) {
        serverLog(LL_VERBOSE,
            "%lu clients connected (%lu slaves), %zu bytes in use",
            listLength(server.clients)-listLength(server.slaves),
            listLength(server.slaves),
            zmalloc_used_memory());
    }
}
...

/* Run the Sentinel timer if we are in sentinel mode. */
run_with_period(100) {
    if (server.sentinel_mode) sentinelTimer();
}
```

第一部分的代码非常简单，每 5000 ms 输出一下日志，打印一下 sentinel 有多少 slave，有多少客户端。

重点来看一下 sentinelTimer 这个函数。
```c
void sentinelTimer(void) {
    sentinelCheckTiltCondition();
    sentinelHandleDictOfRedisInstances(sentinel.masters);
    sentinelRunPendingScripts();
    sentinelCollectTerminatedScripts();
    sentinelKillTimedoutScripts();

    /* We continuously change the frequency of the Redis "timer interrupt"
     * in order to desynchronize every Sentinel from every other.
     * This non-determinism avoids that Sentinels started at the same time
     * exactly continue to stay synchronized asking to be voted at the
     * same time again and again (resulting in nobody likely winning the
     * election because of split brain voting). */
    server.hz = CONFIG_DEFAULT_HZ + rand() % CONFIG_DEFAULT_HZ;
}
```