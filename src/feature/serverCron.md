## ServerCron
> 位于 src/server.c 文件中


### 相关函数
|函数名|说明|
|---|---|
|[serverCron](#serverCron)|时间事件主函数|
|[clientsCron](../func/clientsCron.md)|客户端相关函数|
|[databasesCron](../func/databasesCron.md)|数据库相关函数，用于主动清除过期 key，渐进式 rehash 等|
|[rewriteAppendOnlyFileBackground](./replication.md#startBgsaveForReplication)|重写 AOF日志|
|[freeClientsInAsyncFreeQueue](../func/freeClientsInAsyncFreeQueue.md)|关闭需要异步关闭的客户端|
|[clientsArePaused](../func/clientsArePaused.md)|清除客户端 paused 标记|
|[replicationCron](./replication.md)|主从复制|
|[clusterCron](./cluster.md)|集群模式执行函数|
|[sentinelTimer](./sentinel.md)|sentinel 模式执行函数|
|migrateCloseTimedoutSockets||

### 执行流程

* 更新时间缓存
* 设置一个 LRU 的时钟。
* 记录服务器最大内存使用。
* 记录一些非空数据库的信息（可用键值对数量，已使用键值对数量，带有过期时间的键值对数量）。
* sentinel 模式下，记录一些客户端的连接信息。
* 执行 `clientsCron`，关闭超时客户端等。
* 执行 `databasesCron` 函数，主动清除过期 key，渐进式 rehash，resize 等。
* 检查 bgsave 或 bgrewriteaof 是否执行完成，如果当前没有执行，则判断是否需要执行它们。
* 调用 `freeClientsInAsyncFreeQueue` 关闭需要异步关闭的客户端。
* 调用 `clientsArePaused` 清除客户端 paused 标记。
* 执行主从复制函数 `replicationCron`。
* 如果是集群模式，执行 `clusterCron`。
* 如果是 sentienl 模式，执行 `sentinelTimer`。
* 执行 `migrateCloseTimedoutSockets`。
* 增加 server.cronloops 计数器。

### serverCron

先看一下官方的注释  

* 这是我们的时间中断器，每秒调用 server.hz 次。
* 以下是需要异步执行的操作
    * 主动清除过期的键。
    * 更新软件 watchdog。
    * 更新统计信息。
    * 渐进式 rehash。
    * 触发 BGSAVE 和 AOF 重写。
    * 处理不同种类的客户端超时。
    * 复制。
    * 等等。
* 用到了 run_with_period 这个宏，`run_with_period(milliseconds) { .... }`。

```c
/* This is our timer interrupt, called server.hz times per second.
 * Here is where we do a number of things that need to be done asynchronously.
 * For instance:
 *
 * - Active expired keys collection (it is also performed in a lazy way on
 *   lookup).
 * - Software watchdog.
 * - Update some statistic.
 * - Incremental rehashing of the DBs hash tables.
 * - Triggering BGSAVE / AOF rewrite, and handling of terminated children.
 * - Clients timeout of different kinds.
 * - Replication reconnection.
 * - Many more...
 *
 * Everything directly called here will be called server.hz times per second,
 * so in order to throttle execution of things we want to do less frequently
 * a macro is used: run_with_period(milliseconds) { .... }
 */
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    int j;
    UNUSED(eventLoop);
    UNUSED(id);
    UNUSED(clientData);

    /* Software watchdog: deliver the SIGALRM that will reach the signal
     * handler if we don't return here fast enough. */
    if (server.watchdog_period) watchdogScheduleSignal(server.watchdog_period);

    /* Update the time cache. */
    // 更新时间缓存
    updateCachedTime();

    // 用 run_with_period 这个宏，在指定周期内执行
    run_with_period(100) {
        trackInstantaneousMetric(STATS_METRIC_COMMAND,server.stat_numcommands);
        trackInstantaneousMetric(STATS_METRIC_NET_INPUT,
                server.stat_net_input_bytes);
        trackInstantaneousMetric(STATS_METRIC_NET_OUTPUT,
                server.stat_net_output_bytes);
    }

    /* We have just LRU_BITS bits per object for LRU information.
     * So we use an (eventually wrapping) LRU clock.
     *
     * Note that even if the counter wraps it's not a big problem,
     * everything will still work but some object will appear younger
     * to Redis. However for this to happen a given object should never be
     * touched for all the time needed to the counter to wrap, which is
     * not likely.
     *
     * Note that you can change the resolution altering the
     * LRU_CLOCK_RESOLUTION define. */
    // 设置一个 LRU 时钟
    unsigned long lruclock = getLRUClock();
    atomicSet(server.lruclock,lruclock);

    /* Record the max memory used since the server was started. */
    // 记录服务器最大内存使用
    if (zmalloc_used_memory() > server.stat_peak_memory)
        server.stat_peak_memory = zmalloc_used_memory();

    /* Sample the RSS here since this is a relatively slow call. */
    server.resident_set_size = zmalloc_get_rss();

    /* We received a SIGTERM, shutting down here in a safe way, as it is
     * not ok doing so inside the signal handler. */
    // 收到 SIGTERM  信号，用一种安全的方式关闭
    if (server.shutdown_asap) {
        if (prepareForShutdown(SHUTDOWN_NOFLAGS) == C_OK) exit(0);
        serverLog(LL_WARNING,"SIGTERM received but errors trying to shut down the server, check the logs for more information");
        server.shutdown_asap = 0;
    }

    /* Show some info about non-empty databases */
    // 展示一些非空数据库的信息
    run_with_period(5000) {
        for (j = 0; j < server.dbnum; j++) {
            long long size, used, vkeys;

            // 可用键值对数量
            size = dictSlots(server.db[j].dict);
            // 已经使用键值对数量
            used = dictSize(server.db[j].dict);
            // 带有过期时间的键值对数量
            vkeys = dictSize(server.db[j].expires);
            // 记录一下 log
            if (used || vkeys) {
                serverLog(LL_VERBOSE,"DB %d: %lld keys (%lld volatile) in %lld slots HT.",j,used,vkeys,size);
                /* dictPrintStats(server.dict); */
            }
        }
    }

    /* Show information about connected clients */
    // sentinel 模式下，展示一些客户端的连接信息
    if (!server.sentinel_mode) {
        run_with_period(5000) {
            serverLog(LL_VERBOSE,
                "%lu clients connected (%lu slaves), %zu bytes in use",
                listLength(server.clients)-listLength(server.slaves),
                listLength(server.slaves),
                zmalloc_used_memory());
        }
    }

    /* We need to do a few operations on clients asynchronously. */
    // 对客户端做一些检查，关闭超时的客户端等操作
    clientsCron();

    /* Handle background operations on Redis databases. */
    // 对数据库一些操作，主动清除过期 key，渐进式 rehash，resize 等
    databasesCron();

    /* Start a scheduled AOF rewrite if this was requested by the user while
     * a BGSAVE was in progress. */
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.aof_rewrite_scheduled)
    {
        rewriteAppendOnlyFileBackground();
    }

    /* Check if a background saving or AOF rewrite in progress terminated. */
    // 检查 bgsave 或 bgrewriteaof 是否执行完毕，如果当前没有执行 bgsave，则判断是否需要执行它们
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {
        int statloc;
        pid_t pid;

        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
            int exitcode = WEXITSTATUS(statloc);
            int bysignal = 0;

            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);

            if (pid == -1) {
                serverLog(LL_WARNING,"wait3() returned an error: %s. "
                    "rdb_child_pid = %d, aof_child_pid = %d",
                    strerror(errno),
                    (int) server.rdb_child_pid,
                    (int) server.aof_child_pid);
            } else if (pid == server.rdb_child_pid) {
                backgroundSaveDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else if (pid == server.aof_child_pid) {
                backgroundRewriteDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else {
                if (!ldbRemoveChild(pid)) {
                    serverLog(LL_WARNING,
                        "Warning, detected child with unmatched pid: %ld",
                        (long)pid);
                }
            }
            updateDictResizePolicy();
            closeChildInfoPipe();
        }
    } else {
        /* If there is not a background saving/rewrite in progress check if
         * we have to save/rewrite now */
         for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;

            /* Save if we reached the given amount of changes,
             * the given amount of seconds, and if the latest bgsave was
             * successful or if, in case of an error, at least
             * CONFIG_BGSAVE_RETRY_DELAY seconds already elapsed. */
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK))
            {
                serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                rdbSaveInfo rsi, *rsiptr;
                rsiptr = rdbPopulateSaveInfo(&rsi);
                rdbSaveBackground(server.rdb_filename,rsiptr);
                break;
            }
         }

         /* Trigger an AOF rewrite if needed */
         if (server.rdb_child_pid == -1 &&
             server.aof_child_pid == -1 &&
             server.aof_rewrite_perc &&
             server.aof_current_size > server.aof_rewrite_min_size)
         {
            long long base = server.aof_rewrite_base_size ?
                            server.aof_rewrite_base_size : 1;
            long long growth = (server.aof_current_size*100/base) - 100;
            if (growth >= server.aof_rewrite_perc) {
                serverLog(LL_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                rewriteAppendOnlyFileBackground();
            }
         }
    }


    /* AOF postponed flush: Try at every cron cycle if the slow fsync
     * completed. */
    if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);

    /* AOF write errors: in this case we have a buffer to flush as well and
     * clear the AOF error in case of success to make the DB writable again,
     * however to try every second is enough in case of 'hz' is set to
     * an higher frequency. */
    run_with_period(1000) {
        if (server.aof_last_write_status == C_ERR)
            flushAppendOnlyFile(0);
    }

    /* Close clients that need to be closed asynchronous */
    // 关闭需要异步关闭的客户端
    freeClientsInAsyncFreeQueue();

    /* Clear the paused clients flag if needed. */
    // 清除客户端的 paused 标记
    clientsArePaused(); /* Don't check return value, just use the side effect.*/

    /* Replication cron function -- used to reconnect to master,
     * detect transfer failures, start background RDB transfers and so forth. */
    // 复制函数
    run_with_period(1000) replicationCron();

    /* Run the Redis Cluster cron. */
    // 如果是集群模式下，运行 cluster 相关函数
    run_with_period(100) {
        if (server.cluster_enabled) clusterCron();
    }

    /* Run the Sentinel timer if we are in sentinel mode. */
    // 如果是 sentienl 模式下，执行相关函数
    run_with_period(100) {
        if (server.sentinel_mode) sentinelTimer();
    }

    /* Cleanup expired MIGRATE cached sockets. */
    run_with_period(1000) {
        migrateCloseTimedoutSockets();
    }

    /* Start a scheduled BGSAVE if the corresponding flag is set. This is
     * useful when we are forced to postpone a BGSAVE because an AOF
     * rewrite is in progress.
     *
     * Note: this code must be after the replicationCron() call above so
     * make sure when refactoring this file to keep this order. This is useful
     * because we want to give priority to RDB savings for replication. */
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.rdb_bgsave_scheduled &&
        (server.unixtime-server.lastbgsave_try > CONFIG_BGSAVE_RETRY_DELAY ||
         server.lastbgsave_status == C_OK))
    {
        rdbSaveInfo rsi, *rsiptr;
        rsiptr = rdbPopulateSaveInfo(&rsi);
        if (rdbSaveBackground(server.rdb_filename,rsiptr) == C_OK)
            server.rdb_bgsave_scheduled = 0;
    }

    // 增加计数器
    server.cronloops++;
    return 1000/server.hz;
}
```
