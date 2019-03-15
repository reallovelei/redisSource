## 过期策略
> 位于 src/expire.c

### 存储
redis 中的所有键都保存在 redisDb 中。其中 dict 称为键空间，保存了所有的键值对。过期时间则保存在 expires 中，expires 中的 key 是指向键空间的指针，value 是过期键的到期时间，是一个 unix 时间戳。

##### 数据结构
redis 中的数据都保存在 [redisDb](../struct/common/redisDb.md) 这样的一个数据结构中，其中，和过期相关的主要是 expires 这个字典。这字典中 key 是指向 redis 键空间的一个指针，值是键的过期时间到期的时间戳。

### 策略
* [定时清除](#定时清除)
* [惰性清除](#惰性清除)
* [定期删除](#定期清除)

#### 定时清除
> 在设置 key 的过期时间时, 创建一个时间事件, 由事件负责清除

#### 惰性清除
> 在每次获取 key 时, 检验 key 是否过期, 如果过期, 则清除

在获取任意一个 key 的时候，都会调用一下 db.c 中的 `expireIfNeeded` 函数，来判断 key 是否过期，下面来简单说一下 expireIfNeeded 函数的执行流程。

* 这个函数接收两个参数，一个是数据库的编号，一个是 key，返回 0 表示未过期，返回 1 表示过期。
* 调用 `getExpire` 函数，在 [redisDb](../struct/common/redisDb.md) 的 expires 上，查找该 key 的过期时间，如果没有过期时间，返回 0。
* 如果服务器正在载入 AOF 或 RDB 数据，返回 0，载入过程中不处理。
* 判断当前时间和 key 的过期时间，如果未过期，返回 0。
* `server.stat_expiredkeys++`，更新统计数据。
* 调用 db.c 中的 `propagateExpire` 函数清除 key。
    * 如果开启了 AOF，向 AOF 中追加一条 DEL 的命令。
    * 通知 SLAVE 删除 key。
    * 更新引用计数。
* 通知键空间事件。

#### 定期清除
> 通过定时任务来触发, 具体是 [serverCron](../feature/serverCron.md) 中的 [databasesCron](../func/databasesCron.md) 函数

我们看一下下面这段代码
```c
if (server.active_expire_enabled && server.masterhost == NULL) {
    activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
} else if (server.masterhost != NULL) {
    expireSlaveKeys();
}
```

 如果开启了主动清除且服务器是 master 的情况下，执行 [activeExpireCycle](../func/expire/activeExpireCycle.md)，函数，否则执行 `expireSlaveKeys`。

##### activeExpireCycle
> 尝试去释放一小部分过期的 key，如果过期的 key 很少，这样只会使用很少的 CPU，如果 key 比较多，也可以避免过多的使用内存。

###### 执行流程
1. 如果客户端处于 paused 状态，直接返回。
1. 如果传入的 type 是 ACTIVE_EXPIRE_CYCLE_FAST，判断当前是否已经有快速扫描在执行，执行时间是否超时，这个扫描在一次迭代中只会执行一次，且时间限制在 ACTIVE_EXPIRE_CYCLE_FAST_DURATION 微秒内（默认是 1000）
1. 设置超时时间。
    * 默认为 `timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;`
    * 如果是快速扫描，`timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION;`
1. 遍历每个数据库（默认是 16 个），在`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4`（默认是 5） 个 key 中查找过期的 key。
    * 如果当前数据库中的 expires 字典为空，跳过当前循环。
    * 在 expires 中随机找出一个 key，如果没有过期，则跳过本次循环。
    * 调用 `activeExpireCycleTryExpire` 函数，尝试删除一个过期的 key，删除成功则 `expired++`。
    * 更新一些统计信息。
    * 如果超时，则终止循环。

##### expireSlaveKeys
> 用于处理**可写**的 slave 的过期 key。通常来说，slave 不会处理 key 的过期，它们等待 master 同步 del 命令，但是可写的 slave 是一个例外，如果这个 key 是这个 slave 创建的，那么就需要一种方式来处理它。
* 这个 slave 会定义一个字典 `slaveKeysWithExpire`，来记录这些过期的 key，这个字典会在 `rememberSlaveKeyWithExpire` 函数调用的时候创建，字典的 key 是键，value 是一个 64 位无符号整数。

###### 执行流程
1. 如果 slaveKeysWithExpire 字典为空，直接返回。
1. 开启一个循环，随机从 slaveKeysWithExpire 取出一个。