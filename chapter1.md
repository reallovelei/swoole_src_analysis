# 先从Server开始

我们首先会在cli 模式下创建一个server

```php
$serv = new swoole_server(string $host, int $port, int $mode = SWOOLE_PROCESS,
    int $sock_type = SWOOLE_SOCK_TCP);
```

那么这段代码对应的是

```C
PHP_METHOD(swoole_server, __construct)
{
    zend_size_t host_len = 0;
    char *serv_host;
    long sock_type = SW_SOCK_TCP;
    long serv_port;
    long serv_mode = SW_MODE_PROCESS;

    //only cli env
    if (strcasecmp("cli", sapi_module.name) != 0)
    {
        swoole_php_fatal_error(E_ERROR, "swoole_server must run at php_cli environment.");
        RETURN_FALSE;
    }

    if (SwooleG.main_reactor != NULL)
    {
        swoole_php_fatal_error(E_ERROR, "eventLoop has been created. Unable to create swoole_server.");
        RETURN_FALSE;
    }

    if (SwooleGS->start > 0)
    {
        swoole_php_fatal_error(E_WARNING, "server is already running. Unable to create swoole_server.");
        RETURN_FALSE;
    }

    swServer *serv = sw_malloc(sizeof (swServer));
    swServer_init(serv);

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "sl|ll", &serv_host, &host_len, &serv_port, &serv_mode, &sock_type) == FAILURE)
    {
        swoole_php_fatal_error(E_ERROR, "invalid parameters.");
        return;
    }

#ifdef __CYGWIN__
    serv_mode = SW_MODE_SINGLE;
#elif !defined(SW_USE_THREAD)
    if (serv_mode == SW_MODE_THREAD || serv_mode == SW_MODE_BASE)
    {
        serv_mode = SW_MODE_SINGLE;
        swoole_php_fatal_error(E_WARNING, "PHP can not running at multi-threading. Reset mode to SWOOLE_MODE_BASE");
    }
#endif
    serv->factory_mode = serv_mode;

    if (serv->factory_mode == SW_MODE_SINGLE)
    {
        serv->worker_num = 1;
        serv->max_request = 0;
    }

    bzero(php_sw_server_callbacks, sizeof (zval*) * PHP_SERVER_CALLBACK_NUM);

    swListenPort *port = swServer_add_port(serv, sock_type, serv_host, serv_port);
    if (!port)
    {
        swoole_php_fatal_error(E_ERROR, "listen server port failed.");
        return;
    }

    zval *server_object = getThis();

#ifdef HAVE_PCRE
    zval *connection_iterator_object;
    SW_MAKE_STD_ZVAL(connection_iterator_object);
    object_init_ex(connection_iterator_object, swoole_connection_iterator_class_entry_ptr);
    zend_update_property(swoole_server_class_entry_ptr, server_object, ZEND_STRL("connections"), connection_iterator_object TSRMLS_CC);
#endif

    zend_update_property_stringl(swoole_server_class_entry_ptr, server_object, ZEND_STRL("host"), serv_host, host_len TSRMLS_CC);
    zend_update_property_long(swoole_server_class_entry_ptr, server_object, ZEND_STRL("port"), serv_port TSRMLS_CC);
    zend_update_property_long(swoole_server_class_entry_ptr, server_object, ZEND_STRL("mode"), serv->factory_mode TSRMLS_CC);
    zend_update_property_long(swoole_server_class_entry_ptr, server_object, ZEND_STRL("type"), sock_type TSRMLS_CC);
    swoole_set_object(server_object, serv);

    zval *ports;
    SW_ALLOC_INIT_ZVAL(ports);
    array_init(ports);
    zend_update_property(swoole_server_class_entry_ptr, server_object, ZEND_STRL("ports"), ports TSRMLS_CC);
    server_port_list.zports = ports;

    php_swoole_server_add_port(port TSRMLS_CC);
}
```

其实这段代码本身没什么，先是做了一些判断，然后就是把接收到的参数更新到 server的属性中去了。

主要是

```C
swServer *serv = sw_malloc(sizeof (swServer));
swServer_init(serv);
```

初始化一些server的配置。

```C
void swServer_init(swServer *serv)
{
    swoole_init();
    bzero(serv, sizeof(swServer));

    serv->reactor_num = SW_REACTOR_NUM > SW_REACTOR_MAX_THREAD ? SW_REACTOR_MAX_THREAD : SW_REACTOR_NUM;

    serv->dispatch_mode = SW_DISPATCH_FDMOD;    // 固定worker

    serv->timeout_usec = SW_REACTOR_TIMEO_USEC;  //300ms;
    serv->worker_num = SW_CPU_NUM;               //默认CPU核数
}
```

然后我们会设置 一些配置参数 具体有哪些参数,都什么含义，wiki中都有。

```php
$serv->set(array(
    'worker_num' => 1,
    'daemonize' => 1,
    'buffer_output_size' => 1024 * 1024 *1024,
    //'task_worker_num' => 2,
    //'task_ipc_mode' => 3,
    'max_request' => 0,
    'log_file' => "/var/log/proxy/quote".date('Y-m-d').".log"
));
```

设置完各种回调函数后

最后 我们都会进行一次start

```php
$serv->start()
```



