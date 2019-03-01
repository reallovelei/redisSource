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
* [类型定义](./src/struct/common/aeEventLoop.md)

### 事件类型
#### 文件事件
#### 时间事件

### 生命周期