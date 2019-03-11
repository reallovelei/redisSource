## 管道 pipeline

管道其实并不是 redis 的特性，而是系统提供的一种减少网络请求次数的方式。通过将多个网络请求合并成一个，而提升 IO。

在看 redis 命令执行的时候，可以发现，redis 在执行命令的时候，除了 `multi`, `watch`, `discard`, `exec` 四个命令之外，其他命令都会放入到队列中。
```c
 /* Exec the command */
if (c->flags & CLIENT_MULTI &&
    c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
    c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
{
    queueMultiCommand(c);
    addReply(c,shared.queued);
} else {
    call(c,CMD_CALL_FULL);
    c->woff = server.master_repl_offset;
    if (listLength(server.ready_keys))
        handleClientsBlockedOnLists();
}
```

以 php 为例，我们看一下 php 使用 pipeline 方式是怎么请求 redis 的。
```php
$redis = new Redis();
$redis->connect('localhost',6379);
$pipe = $redis->multi(Redis::PIPELINE);

for($i= 0 ; $i< 10000 ; $i++) {
    // do something
}
$replies = $pipe->exec();
```

可以看到，其实到 redis 这里，也是先开启一个 multi，然后将命令依次放入队列，调用 exec 时，将这一组命令通过一个网络请求放给服务端执行，从而减少了每个命令发送一次网络请求的请求次数。