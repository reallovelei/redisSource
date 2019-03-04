## 配置文件参考

### 引用（INCLUDES）
> redis 可以在主配置文件中引用其他配置文件

|配置项|说明|
|--|--|
|include /path/to/local.conf|引用其他配置文件（后引入的配置会覆盖前面的配置）|

### 模块（MODULES）
> 启动时载入模块

|配置项|说明|
|--|--|
|loadmodule /path/to/my_module.so|启动时载入模块|

### 网络

|配置项|说明|
|--|--|
|bind 192.168.1.100 10.0.0.1|指定监听的 ip|
|protected-mode yes|如果没有设置 bind 和 密码，只允许本地访问（默认开启）|
|port 6379|监听端口号（如果设置 0，redis 将不监听 tcp scoket）|
|tcp-backlog 511|TCP 连接中已完成队列的长度（不能大于 Linux 系统定义 /proc/sys/net/core/somaxconn 中的值）|
|unixsocket /tmp/redis.sock|指定 Unix 套接字的位置，没有设置不监听 Unix 套接字|
|timeout 0|客户端空闲多久后关闭连接（单位：秒），0 为不关闭|
|tcp-keepalive 300|TCP keepalive|

### 通用
|配置项|说明|
|--|--|
|daemonize yes|是否将 redis 作为守护进程运行|
|supervised no|是否通过 upstart 或 systemd 管理守护进程，默认no没有服务监控|
|pidfile /var/run/redis_6379.pid|pid 文件路径（redis 会在启动时创建，退出时删除）|
|loglevel notice|错误级别|
|logfile ""|日志文件名称，空表示标准输出（daemonize 下会发送到 /dev/null）|
|syslog-enabled no|是否启动系统日志记录|
|syslog-ident redis|指定运行用户名|
|syslog-facility local0|指定 syslog 设备|
|databases 16|设置数据库个数|
|always-show-logo yes|启动时是否显示 logo|

### 快照
|配置项|说明|
|--|--|
|save 900 1|900 秒内，有至少 1 个 key 发生改变，就执行持久化|
|stop-writes-on-bgsave-error yes|最后一次 RDB 失败时，是否停止写操作|
|rdbcompression yes|是否开启 RDB 压缩（大约有 10% 的性能消耗，考虑性能可以关掉）|
|rdbchecksum yes|是否检验 RDB 文件校验和|
|dbfilename dump.rdb|RDB 文件名称|
|dir /data/redis|工作目录|

### 同步
|配置项|说明|
|--|--|
|slaveof `<masterip> <masterport>`|主从同步配置|
|masterauth `<master-password>`|master 连接密码|
|slave-serve-stale-data yes|slave 丢失连接时是否继续处理请求|
|slave-read-only yes|slave 是否只读|
|repl-diskless-sync no|（同步策略，磁盘或 socket，默认磁盘）|
|repl-diskless-sync-delay 5|非磁盘同步时，同步延时（单位：秒）|
|repl-ping-slave-period 10|slave 根据指定时间给 master 发送 ping 请求，默认 10 秒|
|repl-timeout 60|同步超时时间|
|repl-disable-tcp-nodelay no|是否在 slave 套接字发送 SYNC 之后禁用 TCP_NODELAY|
|repl-backlog-size 1mb|增量同步设置 backlog 大小|
|repl-backlog-ttl 3600|master 与所有 slave 断开时，多少秒后释放 backlog|
|slave-priority 100|sentinel 选择 slave 升级为 master 的优先级（越小优先级越高）|
|min-slaves-to-write 3 <br> min-slaves-max-lag 10|M 秒内，少于 N 个 slave 连接，master 就停止写操作 <br> 示例：3 秒内少于 10 个 slave 连接，就停止写操作|
|slave-announce-ip <br> slave-announce-port| slave 向 master 声明自己的 ip 和端口|

### 安全
|配置项|说明|
|--|--|
|requirepass password|密码|
|rename-command CONFIG ""|命令重命名（配置为空字符串，表示禁用命令，每行一个命令）|

### 客户端
|配置项|说明|
|--|--|
|maxclients 10000|同时连接最大客户端数量|

### 内存管理
|配置项|说明|
|--|--|
|maxmemory 64000000|内存限制（字节）|
|maxmemory-policy noeviction|内存淘汰策略|
|maxmemory-samples 5|淘汰样本配置，越大越精确，但是更消耗 CPU|

### 惰性清除
|配置项|说明|
|--|--|
|lazyfree-lazy-eviction no <br>lazyfree-lazy-expire no<br>lazyfree-lazy-server-del no<br>slave-lazy-flush no|<br>|

### AOF 配置
|配置项|说明|
|--|--|
|appendonly yes|是否开启 AOF|
|appendfilename "appendonly.aof"|AOF 文件名|
|appendfsync everysec|AOF 写磁盘频率|
|no-appendfsync-on-rewrite no|是否在 BGSAVE 或 BGREWRITEAOF 处理时阻止fsync()|
|auto-aof-rewrite-percentage 100<br>auto-aof-rewrite-min-size 64mb|AOF 重写配置|
|aof-load-truncated yes|因为 AOF 文件被截断产生异常，是否通知用户|
|aof-use-rdb-preamble no|是否开启混合持久化|

### LUA 脚本
|配置项|说明|
|--|--|
|lua-time-limit 5000|lua 脚本超时时间（单位：微秒）|

### 集群
|配置项|说明|
|--|--|
|cluster-enabled yes|是否开启集群|
|cluster-config-file nodes-6379.conf|redis 集群自动生成的配置文件|
|cluster-node-timeout 15000|集群节点超时毫秒数|
|cluster-slave-validity-factor 10|slave 与 master 断开后，指定时间内不成为 master 的系数<br> 公式：(node-timeout * slave-validity-factor) + repl-ping-slave-period|
|cluster-migration-barrier 1|至少有多少个 slave 时，slave 才能成为 master|
|cluster-require-full-coverage yes|如果 redis 集群检测到至少有 1 个 hash slot 不可用，则停止查询服务|
|cluster-slave-no-failover no|<br>|

### 集群 Docker/NAT 支持　
|配置项|说明|
|--|--|
| cluster-announce-ip 10.1.1.5<br>cluster-announce-port 6379<br>cluster-announce-bus-port 6380|<br>|

### 慢查询日志
|配置项|说明|
|--|--|
|slowlog-log-slower-than 10000|超过多少微秒记录慢查询日志|
|slowlog-max-len 128|慢查询日志大小|

### 延时监控
|配置项|说明|
|--|--|
|latency-monitor-threshold 0|单位：微秒，0 表示关闭|

### 键空间通知
|配置项|说明|
|--|--|
|notify-keyspace-events ""|默认处于关闭状态|

### 高级配置
|配置项|说明|
|--|--|
|hash-max-ziplist-entries 512<br>hash-max-ziplist-value 64|ziplist 到 hash 转换的阈值|
|list-max-ziplist-size -2||
|list-compress-depth 0||
|set-max-intset-entries 512||
|zset-max-ziplist-entries 128<br>zset-max-ziplist-value 64||
|hll-sparse-max-bytes 3000||
|activerehashing yes||
|client-output-buffer-limit normal 0 0 0<br>client-output-buffer-limit slave 256mb 64mb 60<br>client-output-buffer-limit pubsub 32mb 8mb 60||
|client-query-buffer-limit 1gb||
|proto-max-bulk-len 512mb||
|hz 10||
|aof-rewrite-incremental-fsync yes||
|lfu-log-factor 10<br>lfu-decay-time 1| <br>|

### ACTIVE DEFRAGMENTATION
|配置项|说明|
|--|--|
|activedefrag yes||
|active-defrag-ignore-bytes 100mb||
|active-defrag-threshold-lower 10||
|active-defrag-threshold-upper 100||
|active-defrag-cycle-min 25||
|active-defrag-cycle-max 75| <br>| 
