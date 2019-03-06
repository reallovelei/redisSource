## databasesCron
> 位于 src/server.c

* 用于处理一些后台的操作，比如主动清除过期的 key，resize 和 渐进式 rehash。

```c
/* This function handles 'background' operations we are required to do
 * incrementally in Redis databases, such as active key expiring, resizing,
 * rehashing. */
void databasesCron(void) {
    /* Expire keys by random sampling. Not required for slaves
     * as master will synthesize DELs for us. */
    // 如果开启了主动清除策略，并且是主服务器
    if (server.active_expire_enabled && server.masterhost == NULL) {
        // 使用 ACTIVE_EXPIRE_CYCLE_SLOW 模式清除
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
    } else if (server.masterhost != NULL) {
        // 检查从主服务器同步过来的 key，并判断是否需要清除
        expireSlaveKeys();
    }

    /* Defrag keys gradually. */
    if (server.active_defrag_enabled)
        activeDefragCycle();

    /* Perform hash tables rehashing if needed, but only if there are no
     * other processes saving the DB on disk. Otherwise rehashing is bad
     * as will cause a lot of copy-on-write of memory pages. */
    // 如果没有进行 bgsave 或者 bgrewriteaof
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
        /* We use global counters so if we stop the computation at a given
         * DB we'll be able to start from the successive in the next
         * cron loop iteration. */
        static unsigned int resize_db = 0;
        static unsigned int rehash_db = 0;
        int dbs_per_call = CRON_DBS_PER_CALL;
        int j;

        /* Don't test more DBs than we have. */
        if (dbs_per_call > server.dbnum) dbs_per_call = server.dbnum;

        /* Resize */
        for (j = 0; j < dbs_per_call; j++) {
            tryResizeHashTables(resize_db % server.dbnum);
            resize_db++;
        }

        /* Rehash */
        // 如果配置中开启了渐进式 rehash，则进行渐进式 rehash
        if (server.activerehashing) {
            for (j = 0; j < dbs_per_call; j++) {
                int work_done = incrementallyRehash(rehash_db % server.dbnum);
                rehash_db++;
                if (work_done) {
                    /* If the function did some work, stop here, we'll do
                     * more at the next cron loop. */
                    break;
                }
            }
        }
    }
}
```