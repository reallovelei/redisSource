## clientsCron
> 位于 src/server.c

```c
#define CLIENTS_CRON_MIN_ITERATIONS 5
void clientsCron(void) {
    /* Make sure to process at least numclients/server.hz of clients
     * per call. Since this function is called server.hz times per second
     * we are sure that in the worst case we process all the clients in 1
     * second. */
    /*
     * 确保每次调用至少处理 numclients/server.hz 个客户端，因为这个函数每秒被调用
     * server.hz 次，所以最坏的情况下，会在一秒处理所有的客户端
     */
    // 连接到服务器的客户端数量
    int numclients = listLength(server.clients);
    // 处理客户端的数量
    int iterations = numclients/server.hz;
    mstime_t now = mstime();

    /* Process at least a few clients while we are at it, even if we need
     * to process less than CLIENTS_CRON_MIN_ITERATIONS to meet our contract
     * of processing each client once per second. */
    // 每次至少处理 CLIENTS_CRON_MIN_ITERATIONS 个客户端
    if (iterations < CLIENTS_CRON_MIN_ITERATIONS)
        iterations = (numclients < CLIENTS_CRON_MIN_ITERATIONS) ?
                     numclients : CLIENTS_CRON_MIN_ITERATIONS;

    while(listLength(server.clients) && iterations--) {
        client *c;
        listNode *head;

        /* Rotate the list, take the current head, process.
         * This way if the client must be removed from the list it's the
         * first element and we don't incur into O(N) computation. */
        // 滚动这个链表，删除尾节点，并把它插入到头节点
        listRotate(server.clients);
        head = listFirst(server.clients);
        c = listNodeValue(head);
        /* The following functions do different service checks on the client.
         * The protocol is that they return non-zero if the client was
         * terminated. */
        // 检验超时的客户端，并关闭它
        if (clientsCronHandleTimeout(c,now)) continue;
        // 调整客户端的 query buffer 大小
        if (clientsCronResizeQueryBuffer(c)) continue;
    }
}
```