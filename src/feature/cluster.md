## 集群
> 位于 src/cluster.c 中

* 启动
* 定时任务（clusterCron）

### 常量定义
```c
#define CLUSTER_DEFAULT_NODE_TIMEOUT 15000
#define CLUSTER_DEFAULT_SLAVE_VALIDITY 10 /* Slave max data age factor. */
#define CLUSTER_DEFAULT_REQUIRE_FULL_COVERAGE 1
#define CLUSTER_FAIL_REPORT_VALIDITY_MULT 2 /* Fail report validity. */
#define CLUSTER_FAIL_UNDO_TIME_MULT 2 /* Undo fail if master is back. */
#define CLUSTER_FAIL_UNDO_TIME_ADD 10 /* Some additional time. */
#define CLUSTER_FAILOVER_DELAY 5 /* Seconds */
#define CLUSTER_DEFAULT_MIGRATION_BARRIER 1
#define CLUSTER_MF_TIMEOUT 5000 /* Milliseconds to do a manual failover. */
#define CLUSTER_MF_PAUSE_MULT 2 /* Master pause manual failover mult. */
#define CLUSTER_SLAVE_MIGRATION_DELAY 5000 /* Delay for slave migration. */
```

### 启动
我们知道，redis 启动分为这么几个步骤。
1. 初始化服务器配置
1. 加载配置文件
1. 初始化服务器
1. 开启事件循环

在这几步中，和集群相关的，主要在**初始服务器配置** 和 **初始化服务器** 两个阶段，当然，加载配置文件的时候，集群的配置也会一起加载。和集群有关的配置，在**初始化服务器**这里已经列出来了。

#### 初始化服务器配置
在初始化服务器配置阶段，可以看到下面的代码。这些代码做了服务器初始化默认配置的工作。
```c
server.cluster_enabled = 0;
server.cluster_node_timeout = CLUSTER_DEFAULT_NODE_TIMEOUT;
server.cluster_migration_barrier = CLUSTER_DEFAULT_MIGRATION_BARRIER;
server.cluster_slave_validity_factor = CLUSTER_DEFAULT_SLAVE_VALIDITY;
server.cluster_require_full_coverage = CLUSTER_DEFAULT_REQUIRE_FULL_COVERAGE;
server.cluster_configfile = zstrdup(CONFIG_DEFAULT_CLUSTER_CONFIG_FILE);
server.cluster_announce_ip = CONFIG_DEFAULT_CLUSTER_ANNOUNCE_IP;
server.cluster_announce_port = CONFIG_DEFAULT_CLUSTER_ANNOUNCE_PORT;
server.cluster_announce_bus_port = CONFIG_DEFAULT_CLUSTER_ANNOUNCE_BUS_PORT;
```

#### 初始化服务器
初始化服务器的时候，会判断是否开启了集群模式，如果开启，则执行集群初始化，代码可以看[这里](../func/clusterInit.md)。
```c
if (server.cluster_enabled) clusterInit();
```

##### 执行流程