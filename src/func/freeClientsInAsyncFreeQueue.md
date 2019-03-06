## freeClientsInAsyncFreeQueue
> 位于 src/networking.c 

```c
void freeClientsInAsyncFreeQueue(void) {
    // 如果有需要关闭的客户端，则执行关闭
    while (listLength(server.clients_to_close)) {
        listNode *ln = listFirst(server.clients_to_close);
        client *c = listNodeValue(ln);

        c->flags &= ~CLIENT_CLOSE_ASAP;
        freeClient(c);
        listDelNode(server.clients_to_close,ln);
    }
}
```