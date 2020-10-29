# Redis消息队列组件 redis-queue

## 说明

  基于Redis的消息队列，支持消息延迟处理。
  
## 项目地址

  https://github.com/walkor/redis-queue
  
## 安装
 
  ```php
  composer require webman/redis-queue
  ```
  
## 配置

新建配置文件 `config/redis_queue.php` 内容类似如下：
  
```php
<?php
return [
    'default' => [
        'host' => 'redis://127.0.0.1:6379',
        'options' => [
            'auth'     => '',
            'database' => 0,
         ]
    ],
];
```

## 投递消息

```php
use Webman\RedisQueue\Client;
// 队列名
$queue = '/topic/send_mail';
// 数据，可以直接传数组，无需序列化
$data = ['to' => 'tom@gmail.com', 'content' => 'hello'];
// 投递消息
Client::send($queue, $data);
// 投递延迟消息，消息会在60秒后处理
Client::send($queue, $data, 60);
```

有时候你需要在其它项目中投递消息并且无法使用`Webman\RedisQueue`，则可以参考以下函数向队列投递消息。

```php
function redis_queue_send($redis, $queue, $data, $delay = 0) {
    $queue_waiting = 'redis-queue-waiting';
    $queue_delay = 'redis-queue-delayed';
    $now = time();
    $package_str = json_encode([
        'id'       => rand(),
        'time'     => $now,
        'delay'    => 0,
        'attempts' => 0,
        'queue'    => $queue,
        'data'     => $data
    ]);
    if ($delay) {
        return $redis->zAdd($queue_delay, $now + $delay, $package_str);
    }
    return $redis->lPush($queue_waiting.$queue, $package_str);
};
```
  
## 消费消息

打开`config/process.php`添加以下配置，增加消费进程

```php
<?php
return [
    ...这里省略了其它配置...
    
    'redis_consumer'  => [
        'handler'     => Webman\RedisQueue\Process\Consumer::class,
        'count'       => 2, // 进程数
        'constructor' => [
            // 消费者类目录
            'consumer_dir' => app_path() . '/queue/redis'
        ]
    ]
];
```

新建 `app/queue/redis/MyMailSend.php` (类名任意)。
```php
<?php

namespace app\queue\redis;

use Webman\RedisQueue\Consumer;

class MailSend implements Consumer
{
    // 队列名
    public $queue = '/topic/send_mail';

    // 连接名，对应 config/redis_queue.php 里的连接`
    public $connection = 'default';

    // 消费
    public function consume($data)
    {
        // 无需反序列化
        var_export($data); // 输出 ['to' => 'tom@gmail.com', 'content' => 'hello']
    }
}
```

  
