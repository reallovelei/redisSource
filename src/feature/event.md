## 事件
> 位于 src/ae.c 文件

redis 中的事件主要分为两类。
* [文件事件](#文件事件)
* [时间事件](#时间事件)

### 事件选择
* redis 会在编译的时候，自动选择最高效的事件处理函数。
```c
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

### 事件状态
```c
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered （目前已注册最大文件描述符的数量）*/
    int setsize; /* max number of file descriptors tracked （已追踪文件描述符的最大数量）*/
    long long timeEventNextId;  /* 时间事件的 id */
    time_t lastTime;     /* Used to detect system clock skew （最后一次执行时间事件的时间）*/
    aeFileEvent *events; /* Registered events （注册的文件时间）*/
    aeFiredEvent *fired; /* Fired events （就绪的文件时间）*/
    aeTimeEvent *timeEventHead; /* 时间事件 */
    int stop; /* 事件处理器的开关 */
    void *apidata; /* This is used for polling API specific data （用于多路复用的特定数据）*/
    aeBeforeSleepProc *beforesleep;    /* 事件处理前执行的函数 */
    aeBeforeSleepProc *aftersleep;     /* 事件处理后执行的函数 */
} aeEventLoop;
```

### 事件类型
#### 定义
```c
#define AE_FILE_EVENTS 1        // 文件事件
#define AE_TIME_EVENTS 2        // 时间事件
#define AE_ALL_EVENTS (AE_FILE_EVENTS|AE_TIME_EVENTS)   // 所有事件
#define AE_DONT_WAIT 4
#define AE_CALL_AFTER_SLEEP 8
```

#### 文件事件
##### 类型

##### 读事件（AE_READABLE 1）
> 当一个新的客户端连接到服务器时，服务器就会为该客户端绑定一个读事件，直到客户端断开连接，这个事件才会被移除。
* 当客户端只连接到服务器，但是并没有向服务器发送命令时，该**客户端的读事件处于等待状态**。
* 当客户端向服务器发送命令，并且请求已经到达（套接字可以无阻塞的执行读操作），该**客户端的读事件处于就绪状态**。

##### 写事件（AE_WRITABLE 2）
> 服务器只有在有命令需要传给客户端时，才会为客户端绑定写事件，在命令传输完成之后，事件就会被移除。
* 当服务器有命令结果需要返回给客户端，但客户端还未能执行无阻塞写，那么**写事件处于等待状态**
* 当服务器有命令结果需要返回给客户端，并且客户端可以进行无阻塞写，那么**写事件处于就绪状态**

##### 数据结构
```c
/* File event structure */
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    aeFileProc *rfileProc;  // 读事件处理器
    aeFileProc *wfileProc;  // 写事件处理器
    void *clientData;       // IO 多路复用私有数据
} aeFileEvent;
```

##### 执行流程
> 文件事件的定义中，定义了两个处理函数，一个读事件处理器，一个写事件处理器。
* 具体的处理函数，要根据 `aeCreateFileEvent` 函数传入的参数而定。
* 如 `aeCreateFileEvent(server.el, fd, AE_READABLE, readQueryFromClient, c)` 会为绑定一个读事件。

#### 时间事件
##### 数据结构
```c
/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. （时间事件 id）*/
    long when_sec; /* seconds （事件到达时间，秒）*/
    long when_ms; /* milliseconds （事件到达时间，毫秒）*/
    aeTimeProc *timeProc;   // 时间处理函数
    aeEventFinalizerProc *finalizerProc;    // 事件释放函数
    void *clientData;       // IO 多路复用私有数据
    struct aeTimeEvent *next;   //  指向下一个时间事件的指针
} aeTimeEvent;
```

##### 执行函数
* [serverCron](./serverCron.md)

##### 执行流程
> 时间事件的处理，是调用 src/ae.c 中的 processTimeEvents 函数执行的。

我们概括一下 [processTimeEvents](../func/event/processTimeEvents.md) 函数的执行过程
* 判断当前时间，如果没有到达下一个事件执行时间，重置时间（作者认为提前处理事件要比延误处理更好）。
* 更新最后一次处理事件的时间。
* 遍历链表，执行到期的事件。
    * 移除将要删除的事件。
    * 跳过本次迭代自己创建的事件。
    * 调用 timeProc 处理事件（这里的 timeProc 是根据创建事件时，传入的参数，目前看只有 [serverCron](./serverCron.md)）。

### 生命周期
在服务器的初始化阶段，最后一步开启了事件循环。
* 注册事件开始执行函数。
* 注册事件结束执行函数。
* 开启事件循环。

```c
aeSetBeforeSleepProc(server.el,beforeSleep);
aeSetAfterSleepProc(server.el,afterSleep);
aeMain(server.el);
aeDeleteEventLoop(server.el);
```

* 我们看一下 `aeMain` 函数（位于 src/ae.c 中）中做了什么。
    * 首先将事件结束开关置为 0（事件类型定义可以参考[这里](../struct/common/aeEventLoop.md)）。
    * 进入事件循环，如果注册了 beforesleep，则执行 beforesleep 函数。
    * 执行 aeProcessEvents 函数处理**所有事件**。
```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```
<br>

看一下 [aeProcessEvents](../func/event/aeProcessEvents.md) 函数定义（位于 src/ae.c 中）。

概括一下执行流程

* 判断是否有事件，如果没有任何事件处理，返回 0。
* 如果当前已注册的事件描述符没有到达最大，**或**有不需要等待的时间事件，则进行。
    * 有时间事件且该事件不需要阻塞，则获取最近的时间事件，将到期时间保存在 timeval 结构中。
    * 如果没有时间事件处理，则根据 AE_DONT_WAIT 设置是否阻塞（flags & AE_DONT_WAIT 为 0 则阻塞）。
    * 调用多路复用 API 获取已就绪的文件事件（根据不同平台不同，可能是 ae_kaueue，ae_epoll 或其他）。
    * 如果有 aftersleep 函数，则执行。
    * 从就绪数组中获取事件。
    * 如果有读事件，处理读事件（mask & AE_READABLE），处理完会将 rfired 置为 1。
    * 如果有写事件，处理写事件（mask & AE_WRITABLE），处理写事件时，会判断 rfired，读写事件只能执行一个。
    * 已处理事件数 +1。
* 处理时间事件。