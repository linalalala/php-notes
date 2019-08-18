# Timer
毫秒精度的定时器。  
官方文档：https://wiki.swoole.com/wiki/page/p-timer.html

## 目录
- [Timer的初步使用](#Timer的初步使用)
- [应用场景](#应用场景)
- [参考](#参考)


### Timer的初步使用
tick：设置一个间隔时钟定时器。
after：在指定的时间后执行函数。
clear：使用定时器ID来删除定时器。
```php
<?php

// 每隔3000毫秒触发一次
$timer_id = Swoole\Timer::tick(3000, function () {
    echo 'tick 3000ms - ' . date('Y-m-d H:i:s') . PHP_EOL;
});

// 9000毫秒后删除定时器
Swoole\Timer::after(9000, function() use ($timer_id) {
    echo 'after 9000ms - ' . date('Y-m-d H:i:s') . PHP_EOL;
    Swoole\Timer::clear($timer_id);
});
```
执行结果：
```bash
[root@99990c8f84e2 swoole]# php Timer_tick.php
tick 3000ms - 2019-08-18 02:00:48
tick 3000ms - 2019-08-18 02:00:51
tick 3000ms - 2019-08-18 02:00:54
after 9000ms - 2019-08-18 02:00:54
```

### 应用场景
每天凌晨定时跑任务脚本，脚本中包含请求第三方的接口，如果接口超时无响应或无数据返回，需要进行重试机制。  
重试机制为：每隔2s重新发送一次请求，但是最多重试5次，该任务脚本5次内成功则停止，第5次仍失败也停止。
```php
<?php

$api_url = 'http://www.baidu.com';
$exec_num = 0;

swoole_timer_tick(2000, function ($timer_id) use ($api_url, &$exec_num) {
    $exec_num ++ ;
    
    echo date('Y-m-d H:i:s') . ' 执行任务中...(' . $exec_num . ')' . PHP_EOL;

    $result = file_get_contents($api_url);
    if ($result) {
        // 删除定时器
        swoole_timer_clear($timer_id);
        echo date('Y-m-d H:i:s') . ' 第（' . $exec_num . '）次请求接口任务执行成功' . PHP_EOL;
    } else {
        if ($exec_num >= 5) {
            // 删除定时器
            swoole_timer_clear($timer_id);
            echo date('Y-m-d H:i:s') . ' 请求接口失败次数达到了5，停止执行' . PHP_EOL;
        } else {
            echo date('Y-m-d H:i:s'). ' 请求接口失败，2秒后再次尝试' . PHP_EOL;
        }
    }
});
```
执行结果：
```bash
2019-08-18 02:18:40 执行任务中...(1)
2019-08-18 02:18:40 第（1）次请求接口任务执行成功
```
若将$api_url换成一个不存在的地址来模拟请求的失败，比如：
```php
$api_url = 'http://url-not-exist.com';
```
则执行结果：
```bash
2019-08-18 02:35:11 执行任务中...(1)
PHP Warning:  file_get_contents(): php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
PHP Warning:  file_get_contents(http://url-not-exist.com): failed to open stream: php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
2019-08-18 02:35:11 请求接口失败，2秒后再次尝试
2019-08-18 02:35:13 执行任务中...(2)
PHP Warning:  file_get_contents(): php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
PHP Warning:  file_get_contents(http://url-not-exist.com): failed to open stream: php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
2019-08-18 02:35:13 请求接口失败，2秒后再次尝试
2019-08-18 02:35:15 执行任务中...(3)
PHP Warning:  file_get_contents(): php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
PHP Warning:  file_get_contents(http://url-not-exist.com): failed to open stream: php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
2019-08-18 02:35:15 请求接口失败，2秒后再次尝试
2019-08-18 02:35:17 执行任务中...(4)
PHP Warning:  file_get_contents(): php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
PHP Warning:  file_get_contents(http://url-not-exist.com): failed to open stream: php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
2019-08-18 02:35:17 请求接口失败，2秒后再次尝试
2019-08-18 02:35:19 执行任务中...(5)
PHP Warning:  file_get_contents(): php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
PHP Warning:  file_get_contents(http://url-not-exist.com): failed to open stream: php_network_getaddresses: getaddrinfo failed: Name or service not known in /root/swoole/Timer_tick.php on line 11
2019-08-18 02:35:19 请求接口失败次数达到了5，停止执行
```

### 参考
- [https://github.com/xinliangnote/Swoole](https://github.com/xinliangnote/Swoole)
