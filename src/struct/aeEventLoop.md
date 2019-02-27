## 事件状态
> 定义位于 src/ae.h 文件中

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