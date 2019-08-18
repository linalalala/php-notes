# Swoole

```php
<?php

// 获取swoole扩展的版本号
var_dump(SWOOLE_VERSION . PHP_EOL);
var_dump(swoole_version() . PHP_EOL);

// 获取本机CPU核数
var_dump(swoole_cpu_num() . PHP_EOL);
```