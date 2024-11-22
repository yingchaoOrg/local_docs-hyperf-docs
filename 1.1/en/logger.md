# 日志

`hyperf/logger` 组件是基于 [psr/logger](https://github.com/php-fig/logger) 实现的，默认使用 [monolog/monolog](https://github.com/Seldaek/monolog) 作为驱动，在 `hyperf-skeleton` 项目内默认提供了一些日志配置，默认使用 `Monolog\Handler\StreamHandler`, 由于 `Swoole` 已经对 `fopen`, `fwrite` 等函数进行了协程化处理，所以只要不将 `useLocking` 参数设置为 `true`，就是协程安全的。

## Installation

```
composer require hyperf/logger
```

## 配置

在 `hyperf-skeleton` 项目内默认提供了一些日志配置，默认情况下，日志的配置文件为 `config/autoload/logger.php` ，示例如下：

```php
<?php

return [
    'default' => [
        'handler' => [
            'class' => \Monolog\Handler\StreamHandler::class,
            'constructor' => [
                'stream' => BASE_PATH . '/runtime/logs/hyperf.log',
                'level' => \Monolog\Logger::DEBUG,
            ],
        ],
        'formatter' => [
            'class' => \Monolog\Formatter\LineFormatter::class,
            'constructor' => [
                'format' => null,
                'dateFormat' => null,
                'allowInlineLineBreaks' => true,
            ]
        ],
    ],
];
```

## 使用

```php
<?php

declare(strict_types=1);

namespace App\Service;

use Psr\Container\ContainerInterface;
use Hyperf\Logger\LoggerFactory;

class DemoService
{
    
    /**
     * @var \Psr\Log\LoggerInterface
     */
    protected $logger;

    public function __construct(LoggerFactory $loggerFactory)
    {
        // default 对应 config/autoload/logger.php 内的 key
        $this->logger = $loggerFactory->get('default');
    }

    public function method()
    {
        // Do somthing.
        $this->logger->info("Your log message.");
    }
}
```

## 关于 monolog 的基础知识

我们结合代码来看一些 `monolog` 中所涉及到的基础概念:

```php
use Monolog\Formatter\LineFormatter;
use Monolog\Handler\FirePHPHandler;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;

// 创建一个 Channel，参数 log 即为 Channel 的名字
$log = new Logger('log');

// 创建两个 Handler，对应变量 $stream 和 $fire
$stream = new StreamHandler('test.log', Logger::WARNING);
$fire = new FirePHPHandler();

// 定义时间格式为 "Y-m-d H:i:s"
$dateFormat = "Y n j, g:i a";
// 定义日志格式为 "[%datetime%] %channel%.%level_name%: %message% %context% %extra%\n"
$output = "%datetime%||%channel||%level_name%||%message%||%context%||%extra%\n";
// 根据 时间格式 和 日志格式，创建一个 Formatter
$formatter = new LineFormatter($output, $dateFormat);

// 将 Formatter 设置到 Handler 里面
$stream->setFormatter($formatter);

// 讲 Handler 推入到 Channel 的 Handler 队列内
$log->pushHandler($stream);
$log->pushHandler($fire);

// clone new log channel
$log2 = $log->withName('log2');

// add records to the log
$log->warning('Foo');

// add extra data to record
// 1. log context
$log->error('a new user', ['username' => 'daydaygo']);
// 2. processor
$log->pushProcessor(function ($record) {
    $record['extra']['dummy'] = 'hello';
    return $record;
});
$log->pushProcessor(new \Monolog\Processor\MemoryPeakUsageProcessor());
$log->alert('czl');
```

- 首先, 实例化一个 `Logger`, 取个名字, 名字对应的就是 `channel`
- 可以为 `Logger` 绑定多个 `Handler`, `Logger` 打日志, 交由 `Handler` 来处理
- `Handler` 可以指定需要处理那些 **日志级别** 的日志, 比如 `Logger::WARNING`, 只处理日志级别 `>=Logger::WARNING` 的日志
- 谁来格式化日志? `Formatter`, 设置好 Formatter 并绑定到相应的 `Handler` 上
- 日志包含那些部分: `"%datetime%||%channel||%level_name%||%message%||%context%||%extra%\n"`
- 区分一下日志中添加的额外信息 `context` 和 `extra`: `context` 由用户打日志时额外指定, 更加灵活; `extra` 由绑定到 `Logger` 上的 `Processor` 固定添加, 比较适合收集一些 **常见信息**

## 更多用法

### 封装 `Log` 类

可能有些时候您更想保持大多数框架使用日志的习惯，那么您可以在 `App` 下创建一个 `Log` 类，并通过 `__callStatic` 魔术方法静态方法调用实现对 `Logger` 的取用以及各个等级的日志记录，我们通过代码来演示一下：

```php
namespace App;

use Hyperf\Logger\Logger;
use Hyperf\Utils\ApplicationContext;

/**
 * @method static Logger get($name)
 * @method static void log($level, $message, array $context = array())
 * @method static void emergency($message, array $context = array())
 * @method static void alert($message, array $context = array())
 * @method static void critical($message, array $context = array())
 * @method static void error($message, array $context = array())
 * @method static void warning($message, array $context = array())
 * @method static void notice($message, array $context = array())
 * @method static void info($message, array $context = array())
 * @method static void debug($message, array $context = array())
 */
class Log
{
    public static function __callStatic($name, $arguments)
    {
        $container = ApplicationContext::getContainer();
        $factory = $container->get(\Hyperf\Logger\LoggerFactory::class);
        if ($name === 'get') {
            return $factory->get(...$arguments);
        }
        $log = $factory->get('default');
        $log->$name(...$arguments);
    }
}
```

默认使用 `default` 的 `Channel` 来记录日志，您也可以通过使用 `Log::get($name)` 方法获得不同 `Channel` 的 `Logger`, 强大的 `容器(Container)` 帮您解决了这一切

### stdout 日志

框架组件所输出的日志在默认情况下是由 `Hyperf\Contract\StdoutLoggerInterface` 接口的实现类 `Hyperf\Framework\Logger\StdoutLogger` 提供支持的，该实现类只是为了将相关的信息通过 `print_r()` 输出在 `标准输出(stdout)`，即为启动 `Hyperf` 的 `终端(Terminal)` 上，也就意味着其实并没有使用到 `monolog` 的，那么如果想要使用 `monolog` 来保持一致要怎么处理呢？

是的, 还是通过强大的 `容器(Container)`.

- 首先, 实现一个 `StdoutLoggerFactory` 类，关于 `Factory` 的用法可在 [依赖注入](zh/di.md) 章节获得更多详细的说明。

```php
<?php
declare(strict_types=1);

namespace App;

use Psr\Container\ContainerInterface;

class StdoutLoggerFactory
{
    public function __invoke(ContainerInterface $container)
    {
        return Log::get('sys');
    }
}
```

- 申明依赖, 使用 `StdoutLoggerInterface` 的地方, 由实际依赖的 `StdoutLoggerFactory` 实例化的类来完成

```php
// config/autoload/dependencies.php
return [
    \Hyperf\Contract\StdoutLoggerInterface::class => \App\StdoutLoggerFactory::class,
];
```

### 不同环境下输出不同格式的日志

上面这么多的使用, 都还只在 monolog 中的 `Logger` 这里打转, 这里来看看 `Handler` 和 `Formatter`

```php
// config/autoload/logger.php
$appEnv = env('APP_ENV', 'dev');
if ($appEnv == 'dev') {
    $formatter = [
        'class' => \Monolog\Formatter\LineFormatter::class,
        'constructor' => [
            'format' => "||%datetime%||%channel%||%level_name%||%message%||%context%||%extra%\n",
            'allowInlineLineBreaks' => true,
            'includeStacktraces' => true,
        ],
    ];
} else {
    $formatter = [
        'class' => \Monolog\Formatter\JsonFormatter::class,
        'constructor' => [],
    ];
}

return [
    'default' => [
        'handler' => [
            'class' => \Monolog\Handler\StreamHandler::class,
            'constructor' => [
                'stream' => 'php://stdout',
                'level' => \Monolog\Logger::INFO,
            ],
        ],
        'formatter' => $formatter,
    ],
]
```

- 默认配置了名为 `default` 的 `Handler`, 并包含了此 `Handler` 及其 `Formatter` 的信息
- 获取 `Logger` 时, 如果没有指定 `Handler`, 底层会自动把 `default` 这一 `Handler` 绑定到 `Logger` 上
- dev(开发)环境: 日志使用 `php://stdout` 输出到 `标准输出(stdout)`, 并且 `Formatter` 中设置 `allowInlineLineBreaks`, 方便查看多行日志
- 非 dev 环境: 日志使用 `JsonFormatter`, 会被格式为 json, 方便投递到第三方日志服务
