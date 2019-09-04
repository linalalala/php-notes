# 基于websocket实现多人聊天室

## 目录
- [预览](#预览)
- [websocket](#websocket)
- [client](#client)
- [server](#server)

### 预览
![websocket](https://raw.githubusercontent.com/duiying/img/master/websocket.png)

### websocket
HTML5开始提供的一种浏览器与服务器进行全双工通讯的网络技术，属于应用层协议。它基于TCP传输协议，并复用HTTP的握手通道。

### client
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <!-- 移动设备优先 -->
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <!-- 新 Bootstrap4 核心 CSS 文件 -->
    <link rel="stylesheet" href="https://cdn.staticfile.org/twitter-bootstrap/4.3.1/css/bootstrap.min.css">
    <!-- jQuery文件。务必在bootstrap.min.js 之前引入 -->
    <script src="https://cdn.staticfile.org/jquery/3.2.1/jquery.min.js"></script>
    <!-- bootstrap.bundle.min.js 用于弹窗、提示、下拉菜单，包含了 popper.min.js -->
    <script src="https://cdn.staticfile.org/popper.js/1.15.0/umd/popper.min.js"></script>
    <!-- 最新的 Bootstrap4 核心 JavaScript 文件 -->
    <script src="https://cdn.staticfile.org/twitter-bootstrap/4.3.1/js/bootstrap.min.js"></script>
</head>
<body>
<nav class="navbar navbar-expand-md navbar-light bg-white shadow-sm">
    <div class="container">
        <a class="navbar-brand" href="javascript:;">
            WebSocket
        </a>
    </div>
</nav>
<div class="container">
    <div class="row">
        <div class="col-sm-4">
            <div class="mt-2">
                <div class="input-group">
                    <input type="text" class="form-control" id="wsUrl" value="ws://106.75.117.140:9501">
                    <button type="button" class="btn btn-success btn-sm ml-1" id="connect" onclick="connect();">连接</button>
                    <button type="button" class="btn btn-danger btn-sm ml-1" id="disconnect" disabled="disabled" onclick="disconnect();">断开</button>
                </div>
                <p><small>格式：ws://IP或域名:端口</small></p>

                <div class="input-group">
                    <textarea name="msg" rows="3" id="msg" class="form-control" placeholder="请输入内容"></textarea>
                </div>

                <div class="form-group">
                    <button type="button" class="btn btn-info btn-block mt-2" onclick="send();" disabled="disabled" id="send">发送</button>
                </div>
            </div>
        </div>
        <div class="col-sm-8">
            <div class="card mt-2">
                <div class="card-header">
                    消息
                </div>
                <div class="card-body" id="websocket-content">

                </div>
            </div>
        </div>
    </div>
</div>
</body>
<script type="text/javascript">
    var websocket;
    var wsUrl;

    function connect()
    {
        try {
            wsUrl = $('#wsUrl').val();
            websocket = new WebSocket(wsUrl);

            websocket.onopen = function(event)
            {
                console.log('客户端与服务端连接成功');
                connectChangeButton();
                alert('连接成功');
            }

            websocket.onmessage = function(event)
            {
                push(event.data);
            }

            websocket.onclose = function(event)
            {
                console.log('连接已关闭');
                disconnectChangeButton();
            }

            websocket.onerror = function(event)
            {
                alert('无法与服务端建立连接');
                console.log('错误：' + event.data);
            }
        } catch (e) {
            alert('无法与服务端建立连接');
        }
    }

    function disconnect()
    {
        websocket.close();
        disconnectChangeButton();
    }

    function push(content)
    {
        if ($('#websocket-content').children('.content-item').length >= 10) {
            $('#websocket-content').children('.content-item:first-child').remove();
        }
        $('#websocket-content').append('<div class="content-item">' + content + '</div>');
    }

    function send()
    {
        var msg = $('#msg').val();
        websocket.send(msg);
        $('#msg').val('');
    }

    function connectChangeButton()
    {
        $('#send').removeAttr('disabled');
        $('#disconnect').removeAttr('disabled');
        $('#connect').attr('disabled', 'disabled');
    }

    function disconnectChangeButton()
    {
        $('#send').attr('disabled', 'disabled');
        $('#connect').removeAttr('disabled');
        $('#disconnect').attr('disabled', 'disabled');
    }
</script>
</html>
```

### server
```php
<?php

class WebSocketServer
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
        $this->server = new swoole_websocket_server($this->host, $this->port);

        // 配置选项，设置运行时的各项参数
        $this->server->set([
            // 启动的worker进程数
            'worker_num' => 2,
            // worker进程的最大任务数
            'max_request' => 4,
            // Task进程的数量
            'task_worker_num' => 4,
            // 数据包分发策略，4是IP分配，保证同一个来源IP的连接数据总会被分配到同一个worker进程
            'dispatch_mode' => 4,
            // 启用守护进程
            'daemonize' => true,
        ]);

        // 注册回调函数
        $this->server->on('Start', [$this, 'onStart']);
        $this->server->on('Open', [$this, 'onOpen']);
        $this->server->on("Message", [$this, 'onMessage']);
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
     * 当WebSocket客户端与服务器建立连接并完成握手后会回调此函数
     *
     * @param $server
     * @param $request
     */
    public function onOpen($server, $request)
    {
        echo '--- onOpen ---' . PHP_EOL;
        echo '客户端：' . $request->fd . ' 已与服务器建立连接并完成握手' . PHP_EOL;
        // 服务端向客户端发送的数据
        $msg = '<div class="text-center"><small class="text-muted">用户' . $request->fd . ' 进入了聊天室</small></div>';
        $server->task($msg);
        echo '--- onOpen end ---' . PHP_EOL;
    }

    /**
     * 当服务器收到来自客户端的数据帧时会回调此函数
     *
     * @param $server
     * @param $frame
     */
    public function onMessage($server, $frame)
    {
        echo '--- onMessage ---' . PHP_EOL;
        echo '客户端：' . $frame->fd . ' 发送数据：' . $frame->data . PHP_EOL;

        // 服务端向客户端发送的数据
        $msg = '<small class="text-success">' . '用户' . $frame->fd . ' ' . date('Y-m-d H:i:s', time()) . '</small><br>' . $frame->data;
        $server->task($msg);

        echo '--- onMessage end ---' . PHP_EOL;
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


        // 向客户端发送数据
        foreach ($server->connections as $fd) {
            $connectionInfo = $server->connection_info($fd);
            // 已握手成功等待浏览器发送数据帧
            if ($connectionInfo['websocket_status'] == WEBSOCKET_STATUS_FRAME) {
                $server->push($fd, $data);
            }
        }

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
     * 客户端连接关闭后，在worker进程中回调此函数
     *
     * @param $server
     * @param $fd
     */
    public function onClose($server, $fd)
    {
        echo '--- onClose ---' . PHP_EOL;
        echo '客户端：' . $fd . ' 关闭连接' . PHP_EOL;
        // 服务端向客户端发送的数据
        $msg = '<div class="text-center"><small class="text-muted">用户' . $fd . ' 退出了聊天室</small></div>';
        $server->task($msg);
        echo '--- onClose end ---' . PHP_EOL;
    }
}

$server = new WebSocketServer();
```