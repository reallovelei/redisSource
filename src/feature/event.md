## 事件
> 位于 src/ae.c 文件

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
```c
#define AE_NONE 0           
#define AE_READABLE 1   // 读事件
#define AE_WRITABLE 2   // 写事件
```

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

我们概括一下 processTimeEvents 函数的执行过程
* 判断当前时间，如果没有到达下一个事件执行时间，重置时间（作者认为提前处理事件要比延误处理更好）。
* 更新最后一次处理事件的时间。
* 遍历链表，执行到期的事件。
    * 移除将要删除的事件。
    * 跳过本次迭代自己创建的事件。
    * 调用 timeProc 处理事件（这里的 timeProc 是根据创建事件时，传入的参数，目前看只有 [serverCron](./serverCron.md)）。

```c
/* Process time events */
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te, *prev;
    long long maxId;
    time_t now = time(NULL);

    /* If the system clock is moved to the future, and then set back to the
     * right value, time events may be delayed in a random way. Often this
     * means that scheduled operations will not be performed soon enough.
     *
     * Here we try to detect system clock skews, and force all the time
     * events to be processed ASAP when this happens: the idea is that
     * processing events earlier is less dangerous than delaying them
     * indefinitely, and practice suggests it is. */
    // 如果没有到达下一个事件执行时间，重置时间
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    // 更新最后一次时间事件执行的时间
    eventLoop->lastTime = now;

    prev = NULL;
    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    // 遍历链表，执行到期的时间事件
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* Remove events scheduled for deletion. */
        // 移除将要被删除的事件
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            if (prev == NULL)
                eventLoop->timeEventHead = te->next;
            else
                prev->next = te->next;
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        /* Make sure we don't process time events created by time events in
         * this iteration. Note that this check is currently useless: we always
         * add new timers on the head, however if we change the implementation
         * detail, this check may be useful again: we keep it here for future
         * defense. */
        // 确保不会执行这次迭代中，自己创建的时间事件
        if (te->id > maxId) {
            te = te->next;
            continue;
        }

        // 获取当前时间
        aeGetTime(&now_sec, &now_ms);
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        prev = te;
        te = te->next;
    }
    return processed;
}
```

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

看一下 aeProcessEvents 函数定义（位于 src/ae.c 中）。

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

```c
/* Process every pending time event, then every pending file event
 * (that may be registered by time event callbacks just processed).
 * Without special flags the function sleeps until some file event
 * fires, or when the next time event occurs (if any).
 *
 * If flags is 0, the function does nothing and returns.
 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
 * if flags has AE_FILE_EVENTS set, file events are processed.
 * if flags has AE_TIME_EVENTS set, time events are processed.
 * if flags has AE_DONT_WAIT set the function returns ASAP until all
 * if flags has AE_CALL_AFTER_SLEEP set, the aftersleep callback is called.
 * the events that's possible to process without to wait are processed.
 *
 * The function returns the number of events processed. */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    // 没有任何事件处理，返回 0
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
    // 当前已注册的事件描述符没有到达上限，或有时间事件且不需要等待。
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        // 获取最近的时间事件
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;

            /* How many milliseconds we need to wait for the next
             * time event to fire? */
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;

            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            // 没有时间事件需要处理，根据 AE_DONT_WAIT 设置是否阻塞
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        /* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
        numevents = aeApiPoll(eventLoop, tvp);

        /* After sleep callback. */
        // 执行 aftersleep 函数
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

	    /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            // 处理读事件
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 处理写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 处理事件数量 +1
            processed++;
        }
    }
    /* Check time events */
    // 执行时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```