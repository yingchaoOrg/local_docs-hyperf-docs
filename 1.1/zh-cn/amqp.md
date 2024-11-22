# AMQP 组件

[hyperf/amqp](https://github.com/hyperf/amqp) 是实现 AMQP 标准的组件，主要适用于对 RabbitMQ 的使用。

## 安装

```bash
composer require hyperf/amqp
```

## 默认配置

|       配置       |  类型  |  默认值   |      备注      |
|:----------------:|:------:|:---------:|:--------------:|
|       host       | string | localhost |      Host      |
|       port       |  int   |   5672    |     端口号     |
|       user       | string |   guest   |     用户名     |
|     password     | string |   guest   |      密码      |
|      vhost       | string |     /     |     vhost      |
| concurrent.limit |  int   |     0     | 同时消费的数量 |
|       pool       | object |           |   连接池配置   |
|      params      | object |           |    基本配置    |

```php
<?php

return [
    'default' => [
        'host' => 'localhost',
        'port' => 5672,
        'user' => 'guest',
        'password' => 'guest',
        'vhost' => '/',
        'concurrent' => [
            'limit' => 1,
        ],
        'pool' => [
            'min_connections' => 1,
            'max_connections' => 10,
            'connect_timeout' => 10.0,
            'wait_timeout' => 3.0,
            'heartbeat' => -1,
        ],
        'params' => [
            'insist' => false,
            'login_method' => 'AMQPLAIN',
            'login_response' => null,
            'locale' => 'en_US',
            'connection_timeout' => 3.0,
            'read_write_timeout' => 6.0,
            'context' => null,
            'keepalive' => false,
            'heartbeat' => 3,
            'close_on_destruct' => false,
        ],
    ],
    'pool2' => [
        ...
    ]
];
```

可在 `producer` 或者 `consumer` 的 `__construct` 函数中, 设置不同 `pool`.

## 投递消息

使用 `gen:producer` 命令创建一个 `producer`

```bash
php bin/hyperf.php gen:amqp-producer DemoProducer
```

在 DemoProducer 文件中，我们可以修改 `@Producer` 注解对应的字段来替换对应的 `exchange` 和 `routingKey`。
其中 `payload` 就是最终投递到消息队列中的数据，所以我们可以随意改写 `__construct` 方法，只要最后赋值 `payload` 即可。
示例如下。

> 使用 `@Producer` 注解时需 `use Hyperf\Amqp\Annotation\Producer;` 命名空间；   

```php
<?php

declare(strict_types=1);

namespace App\Amqp\Producers;

use Hyperf\Amqp\Annotation\Producer;
use Hyperf\Amqp\Message\ProducerMessage;
use App\Models\User;

/**
 * DemoProducer
 * @Producer(exchange="hyperf", routingKey="hyperf")
 */
class DemoProducer extends ProducerMessage
{
    public function __construct($id)
    {
        // 设置不同 pool
        $this->poolName = 'pool2';

        $user = User::where('id', $id)->first();
        $this->payload = [
            'id' => $id,
            'data' => $user->toArray()
        ];
    }
}

```

通过 DI Container 获取 `Hyperf\Amqp\Producer` 实例，即可投递消息。以下实例直接使用 `ApplicationContext` 获取 `Hyperf\Amqp\Producer` 其实并不合理，DI Container 具体使用请到 [依赖注入](zh-cn/di.md) 章节中查看。

```php
<?php
use Hyperf\Amqp\Producer;
use App\Amqp\Producers\DemoProducer;
use Hyperf\Utils\ApplicationContext;

$message = new DemoProducer(1);
$producer = ApplicationContext::getContainer()->get(Producer::class);
$result = $producer->produce($message);

```

## 消费消息

使用 `gen:amqp-consumer` 命令创建一个 `consumer`。

```bash
php bin/hyperf.php gen:amqp-consumer DemoConsumer
```

在 DemoConsumer 文件中，我们可以修改 `@Consumer` 注解对应的字段来替换对应的 `exchange`、`routingKey` 和 `queue`。
其中 `$data` 就是解析后的消息数据。
示例如下。

> 使用 `@Consumer` 注解时需 `use Hyperf\Amqp\Annotation\Consumer;` 命名空间；   

```php
<?php

declare(strict_types=1);

namespace App\Amqp\Consumers;

use Hyperf\Amqp\Annotation\Consumer;
use Hyperf\Amqp\Message\ConsumerMessage;
use Hyperf\Amqp\Result;

/**
 * @Consumer(exchange="hyperf", routingKey="hyperf", queue="hyperf", nums=1)
 */
class DemoConsumer extends ConsumerMessage
{
    public function consume($data): string
    {
        print_r($data);
        return Result::ACK;
    }
}
```

### 禁止消费进程自启

默认情况下，使用了 `@Consumer` 注解后，框架会自动创建子进程启动消费者，并且会在子进程异常退出后，重新拉起。
如果出于开发阶段，进行消费者调试时，可能会因为消费其他消息而导致调试不便。

这种情况，只需要在 `@Consumer` 注解中配置 `enable=false` (默认为 `true` 跟随服务启动)或者在对应的消费者中重写类方法 `isEnable()` 返回 `false` 即可

```php
<?php

declare(strict_types=1);

namespace App\Amqp\Consumers;

use Hyperf\Amqp\Annotation\Consumer;
use Hyperf\Amqp\Message\ConsumerMessage;
use Hyperf\Amqp\Result;

/**
 * @Consumer(exchange="hyperf", routingKey="hyperf", queue="hyperf", nums=1, enable=false)
 */
class DemoConsumer extends ConsumerMessage
{
    public function consume($data): string
    {
        print_r($data);
        return Result::ACK;
    }

    public function isEnable(): bool
    {
        return parent::isEnable();
    }
}
```

### 设置最大消费数

可以修改 `@Consumer` 注解中的 `maxConsumption` 属性，设置此消费者最大处理的消息数，达到指定消费数后，消费者进程会重启。

### 消费结果

框架会根据 `Consumer` 内的 `consume` 方法所返回的结果来决定该消息的响应行为，共有 4 中响应结果，分别为 `\Hyperf\Amqp\Result::ACK`、`\Hyperf\Amqp\Result::NACK`、`\Hyperf\Amqp\Result::REQUEUE`、`\Hyperf\Amqp\Result::DROP`，每个返回值分别代表如下行为：

| 返回值                       | 行为                                                                 |
|------------------------------|----------------------------------------------------------------------|
| \Hyperf\Amqp\Result::ACK     | 确认消息正确被消费掉了                                               |
| \Hyperf\Amqp\Result::NACK    | 消息没有被正确消费掉，以 `basic_nack` 方法来响应                     |
| \Hyperf\Amqp\Result::REQUEUE | 消息没有被正确消费掉，以 `basic_reject` 方法来响应，并使消息重新入列 |
| \Hyperf\Amqp\Result::DROP    | 消息没有被正确消费掉，以 `basic_reject` 方法来响应                   |

## RPC 远程过程调用

除了典型的消息队列场景，我们还可以通过 AMQP 来实现 RPC 远程过程调用，本组件也为这个实现提供了对应的支持。

### 创建消费者

RPC 使用的消费者，与典型消息队列场景的消费者实现基本无差，唯一的区别是需要通过调用 `reply` 方法返回数据给生产者。

```php
<?php

declare(strict_types=1);

namespace App\Amqp\Consumer;

use Hyperf\Amqp\Annotation\Consumer;
use Hyperf\Amqp\Message\ConsumerMessage;
use Hyperf\Amqp\Result;
use PhpAmqpLib\Message\AMQPMessage;

/**
 * @Consumer(exchange="hyperf", routingKey="hyperf", queue="rpc.reply", name="ReplyConsumer", nums=1, enable=true)
 */
class ReplyConsumer extends ConsumerMessage
{
    public function consumeMessage($data, AMQPMessage $message): string
    {
        $data['message'] .= 'Reply:' . $data['message'];

        $this->reply($data, $message);

        return Result::ACK;
    }
}
```

### 发起 RPC 调用

作为生成者发起一次 RPC 远程过程调用也非常的简单，只需通过依赖注入容器获得 `Hyperf\Amqp\RpcClient` 对象并调用其中的 `call` 方法即可，返回的结果是消费者 reply 的数据，如下所示：

```php
<?php
use Hyperf\Amqp\Message\DynamicRpcMessage;
use Hyperf\Amqp\RpcClient;
use Hyperf\Utils\ApplicationContext;

$rpcClient = ApplicationContext::getContainer()->get(RpcClient::class);
// 在 DynamicRpcMessage 上设置与 Consumer 一致的 Exchange 和 RoutingKey
$result = $rpcClient->call(new DynamicRpcMessage('hyperf', 'hyperf', ['message' => 'Hello Hyperf'])); 

// $result:
// array(1) {
//     ["message"]=>
//     string(18) "Reply:Hello Hyperf"
// }
```

### 抽象 RpcMessage

上面的 RPC 调用过程是直接通过 `Hyperf\Amqp\Message\DynamicRpcMessage` 类来完成 Exchange 和 RoutingKey 的定义，并传递消息数据，在生产项目的设计上，我们可以对 RpcMessage 进行一层抽象，以统一 Exchange 和 RoutingKey 的定义。   

我们可以创建对应的 RpcMessage 类如 `App\Amqp\FooRpcMessage` 如下：

```php
<?php
use Hyperf\Amqp\Message\RpcMessage;

class FooRpcMessage extends RpcMessage
{

    protected $exchange = 'hyperf';

    protected $routingKey = 'hyperf';
    
    public function __construct($data)
    {
        // 要传递数据
        $this->payload = $data;
    }

}
```

这样我们进行 RPC 调用时，只需直接传递 `FooRpcMessage` 实例到 `call` 方法即可，无需每次调用时都去定义 Exchange 和 RoutingKey。
