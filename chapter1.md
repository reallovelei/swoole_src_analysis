# 先从Server开始

我们首先会在cli 模式下创建一个server

```
$serv = new swoole_server(string $host, int $port, int $mode = SWOOLE_PROCESS,
    int $sock_type = SWOOLE_SOCK_TCP);
```



