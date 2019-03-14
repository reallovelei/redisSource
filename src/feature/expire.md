## 过期策略
> 位于 src/expire.c

### 存储
redis 中的所有键都保存在 redisDb 中。其中 dict 称为键空间，保存了所有的键值对。过期时间则保存在 expires 中，expires 中的 key 是指向键空间的指针，value 是过期键的到期时间，是一个 unix 时间戳。

##### 数据结构
redis 中的数据都保存在 [redisDb](../struct/common/redisDb.md) 这样的一个数据结构中，其中，和过期相关的主要是 expires 这个字典。这字典中 key 是指向 redis 键空间的一个指针，值是键的过期时间到期的时间戳。

### 策略
* 定时清除
* 惰性清除
* 定期删除

#### 定时清除
> 在设置 key 的过期时间时, 创建一个时间事件, 由事件负责清除

#### 惰性清除
> 在每次获取 key 时, 检验 key 是否过期, 如果过期, 则清除

#### 定期清除
> 通过定时任务来触发, 具体是 serverCron 中的 databasesCron 函数