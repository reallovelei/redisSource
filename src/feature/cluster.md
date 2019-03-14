## 集群
> 位于 src/cluster.c 中，客户端通过在本地缓存一份槽位映射表，来实现快速定位目标节点。

* [启动](#启动)
* [定时任务](#定时任务)
* [迁移](#迁移)
* [命令执行](#命令执行)

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

其实，和集群相关的配置还不止上面这些，上面的只是在 server 中列出来的部分，还有一部分在 cluster 中列出的。
相关定义可以看[这里](../struct/common/cluster.md)

#### 初始化服务器
初始化服务器的时候，会判断是否开启了集群模式，如果开启，则执行集群初始化。
```c
if (server.cluster_enabled) clusterInit();
```
代码可以看[这里](../func/cluster/clusterInit.md)。

##### 执行流程
> 集群中有两个常用的数据结构
> [clusterNode](../struct/common/cluster.md#集群节点定义) 类型的 myself，用于表示自身节点。
> [clusterState](../struct/common/cluster.md#集群状态定义) 类型的 server.cluster，用于表示集群节点状态。

* 初始化集群配置。
    * 在 clusterInit 函数中，首先创建了一个 clusterState 的对象，把它存在了 server.cluster 中。
    * 之后会初始化集群节点，其实就是创建了一个字典对象。
* 载入或者创建每个 node 的配置文件。
    * 这一步会先调用 [createClusterNode](../func/cluster/createClusterNode.md) 函数创建一个集群节点，函数会返回一个 clusterNode 类型的数据
    * 之后调用 `clusterAddNode` 将节点加入到 `server.cluster->nodes` 中。
* 为 cluster 创建一个文件事件 clusterAcceptHandler（具体说应该是一个读事件），用于 accept 连接请求。
* 调用 `raxNew()` 函数初始化（slot，slots_to_keys 是一个基数树）。
* 设置 myself 的 port 和 cport。
    * 如果 server.cluster_announce_port 存 `port =  server.cluster_announce_port`
    * 如果 server.cluster_announce_bus_port 存在，`cport = server.cluster_announce_bus_port`
* 调用 `resetManualFailover` 设置手动故障转移参数。

### 定时任务
> 集群的定时任务，是通过 [clusterCron](../func/cluster/clusterCron.md) 函数执行的，这个函数每秒会执行 10 次（clusterCron 会在 serverCron 触发）。

#### 执行流程

* 向所有断开连接的节点和未连接的节点发送连接请求。
* 每执行 10 次随机向一些节点发送 ping 请求。
* 遍历所有节点，检查是否需要将某节点标记为下线。
* 如果是从节点，且没有开启复制，在我们知道 master 的情况下，开启复制。
* 更新集群状态。

### 迁移
redis 迁移一般是使用工具 `redis-trib` 来完成，本质上是调用了 `migrate` 命令来执行的，`migrate` 命令对应的函数位于 `src/cluster.c` 中的 `migrateCommand`。

### 命令执行
> 集群的命令执行，和普通模式下，主要多了寻找节点的步骤。

在 server.c 文件的 [processCommand](../func/server/processCommand.md) 函数中，可以找到下面这段代码。
```c
/* If cluster is enabled perform the cluster redirection here.
    * However we don't perform the redirection if:
    * 1) The sender of this command is our master.
    * 2) The command has no key arguments. */
if (server.cluster_enabled &&
    !(c->flags & CLIENT_MASTER) &&
    !(c->flags & CLIENT_LUA &&
        server.lua_caller->flags & CLIENT_MASTER) &&
    !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
        c->cmd->proc != execCommand))
{
    int hashslot;
    int error_code;
    clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,
                                    &hashslot,&error_code);
    if (n == NULL || n != server.cluster->myself) {
        if (c->cmd->proc == execCommand) {
            discardTransaction(c);
        } else {
            flagTransaction(c);
        }
        clusterRedirectClient(c,n,hashslot,error_code);
        return C_OK;
    }
}
```

下面我们来分析一下集群模式下的执行逻辑。
* 如果开启了集群模式，则执行下面逻辑。
* 调用 [getNodeByQuery](../func/cluster/getNodeByQuery.md) 函数获取集群节点。
* 如果没有获取到节点，或者获取到的节点不是本节点，判断当前命令是否是 exec。
    * 如果是 exec 命令，放弃当前事务。
    * 如果不是 exec 命令，标记当前数据为 dirty。
    * 调用 `clusterRedirectClient` 函数，返回一条 move 指令。
