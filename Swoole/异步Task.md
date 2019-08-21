# 异步Task

## 目录
- [介绍](#介绍)
- [应用场景](#应用场景)
  - [执行结果](#执行结果)
  - [Server.php](#Server.php)
  - [Client.php](#Client.php)

### 介绍
Swoole提供了异步任务处理的功能，可以投递一个异步任务到TaskWorker进程池中执行，不影响当前请求的处理速度。  
常用的场景：异步支付处理、异步订单处理、异步日志处理、异步发送邮件/短信等。  
官方文档：https://wiki.swoole.com/wiki/page/481.html、https://wiki.swoole.com/wiki/page/134.html

### 应用场景
模拟发短信的场景：用户点击发送短信，后端投递一个异步任务并立即给用户返回已发送短信的提示信息，用户无需等待真实发送短信的耗时。当异步真正地发送完短信后，执行回调函数。

#### 执行结果
![Task](https://raw.githubusercontent.com/duiying/img/master/Task.png)

#### Server.php
```php
<?php

/**
 * TCP服务端
 */
Class Server
{
    // Server对象
    private $server;

    // 监听地址，0.0.0.0表示监听全部地址
    private $host = '0.0.0.0';

    // 监听端口
    private $port = 9501;

    public function __construct()
    {
        // 创建一个TCP Server对象
        $this->server = new swoole_server($this->host, $this->port);

        // 配置选项，设置运行时的各项参数
        $this->server->set([
            // 启动的worker进程数
            'worker_num' => 2,
            // worker进程的最大任务数
            'max_request' => 4,
            // Task进程的数量
            'task_worker_num' => 4,
            // 数据包分发策略，2是固定模式，根据连接的文件描述符分配worker
            'dispatch_mode' => 2,
        ]);

        // 注册回调函数
        $this->server->on('Start', [$this, 'onStart']);
        $this->server->on('Connect', [$this, 'onConnect']);
        $this->server->on("Receive", [$this, 'onReceive']);
        $this->server->on("Close", [$this, 'onClose']);
        $this->server->on("Task", [$this, 'onTask']);
        $this->server->on("Finish", [$this, 'onFinish']);

        // 启动服务
        $this->server->start();
    }

    /**
     * 启动后在主进程(master)的主线程回调此函数
     *
     * @param $server
     */
    public function onStart($server)
    {
        echo '--- onStart ---' . PHP_EOL;
        echo 'swoole V' . SWOOLE_VERSION . ' 服务已启动' . PHP_EOL;
        // 当前服务主进程的PID
        echo 'master_pid：' . $server->master_pid . PHP_EOL;
        // 当前服务管理进程的PID
        echo 'manager_pid：' . $server->manager_pid . PHP_EOL;
        echo "--- onStart end ---" . PHP_EOL;
    }

    /**
     * 新的连接进入时，在worker进程中回调
     *
     * @param $server
     * @param $fd
     */
    public function onConnect($server, $fd)
    {
        echo '--- onConnect ---' . PHP_EOL;
        echo '客户端：' . $fd . ' 已连接' . PHP_EOL;
        echo '--- onConnect end ---' . PHP_EOL;
    }

    /**
     * 接收到数据时回调此函数，发生在worker进程中
     *
     * @param $server
     * @param $fd
     * @param $reactor_id
     * @param $data
     */
    public function onReceive($server, $fd, $reactor_id, $data)
    {
        echo '--- onReceive ---' . PHP_EOL;
        // 当前进程的操作系统进程ID
        echo 'worker_pid：' . $server->worker_pid . PHP_EOL;
        echo 'data：' . $data . PHP_EOL;
        $param = [
            'fd' => $fd,
            'data' => $data
        ];

        // 投递一个异步任务到task_worker池中
        $res = $server->task(json_encode($param));
        if ($res === false) {
            echo '任务投递失败' . PHP_EOL;
            return;
        }

        $server->send($fd, '已发送短信，请注意查收。');
        echo '--- onReceive end ---' . PHP_EOL;
    }

    /**
     * 在task_worker进程内被调用
     *
     * @param $server
     * @param $task_id
     * @param $src_worker_id
     * @param $data
     */
    public function onTask($server, $task_id, $src_worker_id, $data)
    {
        echo '--- onTask ---' . PHP_EOL;
        // 当前worker进程或Task进程的编号
        echo 'worker_id：' . $server->worker_id . PHP_EOL;
        // 当前进程的操作系统进程ID
        echo 'worker_pid：' . $server->worker_pid . PHP_EOL;
        // 任务ID
        echo 'task_id：' . $task_id . PHP_EOL;

        // 发短信业务代码
        echo '发短信中...' . PHP_EOL;
        sleep(5);
        echo '发短信结束' . PHP_EOL;

        // 向客户端发送数据
        $data_arr = json_decode($data, true);
        $server->send($data_arr['fd'], 'data：' . $data_arr['data']);

        // finish用于在Task进程中通知worker进程，投递的任务已完成
        $server->finish($data);

        echo '--- onTask end ---' . PHP_EOL;
    }

    /**
     * 当worker进程投递的任务在task_worker中完成时，task进程会通过finish方法将任务处理的结果发送给worker进程
     *
     * @param $server
     * @param $task_id
     * @param $data
     */
    public function onFinish($server, $task_id, $data)
    {
        echo '--- onFinish ---' . PHP_EOL;
        echo 'Task：' . $task_id . '已完成' . PHP_EOL;
        echo 'worker_pid：' . $server->worker_pid . PHP_EOL;
        echo '--- onFinish end ---' . PHP_EOL;
    }

    /**
     * TCP客户端连接关闭后，在worker进程中回调此函数
     *
     * @param $server
     * @param $fd
     */
    public function onClose($server, $fd)
    {
        echo 'client：' . $fd . ' closed' . PHP_EOL;
    }
}

$server = new Server();
```

#### Client.php
```php
<?php

/**
 * TCP客户端
 */
class Client
{
    private $client;

    // 服务端地址
    private $host = '127.0.0.1';

    // 服务端端口
    private $port = 9501;

    public function __construct()
    {
        // 创建TCP客户端
        $this->client = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_ASYNC);

        // 注册回调函数
        $this->client->on('Connect', [$this, 'onConnect']);
        $this->client->on('Receive', [$this, 'onReceive']);
        $this->client->on('Close', [$this, 'onClose']);
        $this->client->on('Error', [$this, 'onError']);
    }

    public function connect()
    {
        if (!$fp = $this->client->connect($this->host, $this->port , 1)) {
            echo 'errMsg：' . $fp->errMsg . PHP_EOL;
            echo 'errCode：' . $fp->errCode . PHP_EOL;
            return;
        }
    }

    /**
     * 客户端连接服务器成功后会回调此函数
     *
     * @param $client
     */
    public function onConnect($client)
    {
        fwrite(STDOUT, '请输入手机号：');
        $msg = trim(fgets(STDIN));
        $this->send($msg);
    }

    /**
     * 客户端收到来自于服务器端的数据时会回调此函数
     *
     * @param $client
     * @param $data
     */
    public function onReceive($client, $data)
    {
        echo PHP_EOL . 'received：' . $data . PHP_EOL;
    }

    /**
     * 发送数据到远程服务器
     *
     * @param $data
     */
    public function send($data)
    {
        $this->client->send($data);
    }

    /**
     * 连接被关闭时回调此函数
     *
     * @param $client
     */
    public function onClose($client)
    {
        echo 'client closed connection' . PHP_EOL;
    }

    /**
     * 连接服务器失败时会回调此函数
     *
     * @param $client
     */
    public function onError($client)
    {
        echo 'client connect failed' . PHP_EOL;
    }
}

$client = new Client();
$client->connect();
```