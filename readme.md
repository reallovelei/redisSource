## Redis 源码分析
> 基于 4.0.2 版本

### 目录结构
redis 的代码大都位于 src 目录下。

### 数据结构
#### 常见结构体定义
* [服务器定义 redisServer](./src/struct/redisServer.md)
* [数据库定义 redisDb](./src/struct/redisDb.md)
* [客户端定义 client](./src/struct/client.md)
* [事件处理器定义 aeEventLoop](./src/struct/aeEventLoop.md)

#### 基本数据类型
* sds
* list
* dict

### 启动过程
#### 服务端
> 入口文件位于 server.c 文件，main 函数。

##### 生命周期
整个生命周期可以概括为以下 4 个步骤。
1. [初始化服务器配置](./src/server/初始化服务器配置.md)
1. [加载配置文件](./src/server/读取配置文件.md)
1. [初始化服务器](./src/server/初始化服务器.md)
1. [开启事件循环](./src/server/开启事件循环.md)

对应代码来看
```c
int main(int argc, char **argv) {
    struct timeval tv;
    int j;
    ...
    // 初始化服务器配置
    initServerConfig();
    ...
    // 读取配置文件
    if (argc >= 2) {
        ...
    }
    ...
    // 初始化服务器
    initServer();
    ...
    // 开启事件循环
    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeSetAfterSleepProc(server.el,afterSleep);
    aeMain(server.el);
    aeDeleteEventLoop(server.el);
    // 结束
    return 0;
}
```

#### 客户端
##### 建立链接
##### 一次完整的命令请求

### 功能实现