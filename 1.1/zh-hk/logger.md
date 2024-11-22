# 日誌

`hyperf/logger` 組件是基於 [psr/logger](https://github.com/php-fig/log) 實現的，默認使用 [monolog/monolog](https://github.com/Seldaek/monolog) 作為驅動，在 `hyperf-skeleton` 項目內默認提供了一些日誌配置，默認使用 `Monolog\Handler\StreamHandler`, 由於 `Swoole` 已經對 `fopen`, `fwrite` 等函數進行了協程化處理，所以只要不將 `useLocking` 參數設置為 `true`，就是協程安全的。

## 安裝

```
composer require hyperf/logger
```

## 配置

在 `hyperf-skeleton` 項目內默認提供了一些日誌配置，默認情況下，日誌的配置文件為 `config/autoload/logger.php` ，示例如下：

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
        // 第一個參數對應日誌的 name, 第二個參數對應 config/autoload/logger.php 內的 key
        $this->logger = $loggerFactory->get('log', 'default');
    }

    public function method()
    {
        // Do somthing.
        $this->logger->info("Your log message.");
    }
}
```

## 關於 monolog 的基礎知識

我們結合代碼來看一些 `monolog` 中所涉及到的基礎概念:

```php
use Monolog\Formatter\LineFormatter;
use Monolog\Handler\FirePHPHandler;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;

// 創建一個 Channel，參數 log 即為 Channel 的名字
$log = new Logger('log');

// 創建兩個 Handler，對應變量 $stream 和 $fire
$stream = new StreamHandler('test.log', Logger::WARNING);
$fire = new FirePHPHandler();

// 定義時間格式為 "Y-m-d H:i:s"
$dateFormat = "Y n j, g:i a";
// 定義日誌格式為 "[%datetime%] %channel%.%level_name%: %message% %context% %extra%\n"
$output = "%datetime%||%channel||%level_name%||%message%||%context%||%extra%\n";
// 根據 時間格式 和 日誌格式，創建一個 Formatter
$formatter = new LineFormatter($output, $dateFormat);

// 將 Formatter 設置到 Handler 裏面
$stream->setFormatter($formatter);

// 將 Handler 推入到 Channel 的 Handler 隊列內
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

- 首先, 實例化一個 `Logger`, 取個名字, 名字對應的就是 `channel`
- 可以為 `Logger` 綁定多個 `Handler`, `Logger` 打日誌, 交由 `Handler` 來處理
- `Handler` 可以指定需要處理哪些 **日誌級別** 的日誌, 比如 `Logger::WARNING`, 只處理日誌級別 `>=Logger::WARNING` 的日誌
- 誰來格式化日誌? `Formatter`, 設置好 Formatter 並綁定到相應的 `Handler` 上
- 日誌包含哪些部分: `"%datetime%||%channel||%level_name%||%message%||%context%||%extra%\n"`
- 區分一下日誌中添加的額外信息 `context` 和 `extra`: `context` 由用户打日誌時額外指定, 更加靈活; `extra` 由綁定到 `Logger` 上的 `Processor` 固定添加, 比較適合收集一些 **常見信息**

## 更多用法

### 封裝 `Log` 類

可能有些時候您更想保持大多數框架使用日誌的習慣，那麼您可以在 `App` 下創建一個 `Log` 類，並通過 `__callStatic` 魔術方法靜態方法調用實現對 `Logger` 的取用以及各個等級的日誌記錄，我們通過代碼來演示一下：

```php
namespace App;

use Hyperf\Logger\Logger;
use Hyperf\Utils\ApplicationContext;

class Log
{
    public static function get(string $name = 'app')
    {
        return ApplicationContext::getContainer()->get(\Hyperf\Logger\LoggerFactory::class)->get($name);
    }
}
```

默認使用 `Channel` 名為 `app` 來記錄日誌，您也可以通過使用 `Log::get($name)` 方法獲得不同 `Channel` 的 `Logger`, 強大的 `容器(Container)` 幫您解決了這一切

### stdout 日誌

框架組件所輸出的日誌在默認情況下是由 `Hyperf\Contract\StdoutLoggerInterface` 接口的實現類 `Hyperf\Framework\Logger\StdoutLogger` 提供支持的，該實現類只是為了將相關的信息通過 `print_r()` 輸出在 `標準輸出(stdout)`，即為啟動 `Hyperf` 的 `終端(Terminal)` 上，也就意味着其實並沒有使用到 `monolog` 的，那麼如果想要使用 `monolog` 來保持一致要怎麼處理呢？

是的, 還是通過強大的 `容器(Container)`.

- 首先, 實現一個 `StdoutLoggerFactory` 類，關於 `Factory` 的用法可在 [依賴注入](zh-hk/di.md) 章節獲得更多詳細的説明。

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

- 申明依賴, 使用 `StdoutLoggerInterface` 的地方, 由實際依賴的 `StdoutLoggerFactory` 實例化的類來完成

```php
// config/autoload/dependencies.php
return [
    \Hyperf\Contract\StdoutLoggerInterface::class => \App\StdoutLoggerFactory::class,
];
```

### 不同環境下輸出不同格式的日誌

上面這麼多的使用, 都還只在 monolog 中的 `Logger` 這裏打轉, 這裏來看看 `Handler` 和 `Formatter`

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

- 默認配置了名為 `default` 的 `Handler`, 幷包含了此 `Handler` 及其 `Formatter` 的信息
- 獲取 `Logger` 時, 如果沒有指定 `Handler`, 底層會自動把 `default` 這一 `Handler` 綁定到 `Logger` 上
- dev(開發)環境: 日誌使用 `php://stdout` 輸出到 `標準輸出(stdout)`, 並且 `Formatter` 中設置 `allowInlineLineBreaks`, 方便查看多行日誌
- 非 dev 環境: 日誌使用 `JsonFormatter`, 會被格式為 `json`, 方便投遞到第三方日誌服務

### 日誌文件按日期輪轉

如果您希望日誌文件可以按照日期輪轉，可以通過 `Mongolog` 已經提供了的 `Monolog\Handler\RotatingFileHandler` 來實現，配置如下：

修改 `config/autoload/logger.php` 配置文件，將 `Handler` 改為 `Monolog\Handler\RotatingFileHandler::class`，並將 `stream` 字段改為 `filename` 即可。

```php
<?php

return [
    'default' => [
        'handler' => [
            'class' => Monolog\Handler\RotatingFileHandler::class,
            'constructor' => [
                'filename' => BASE_PATH . '/runtime/logs/hyperf.log',
                'level' => Monolog\Logger::DEBUG,
            ],
        ],
        'formatter' => [
            'class' => Monolog\Formatter\LineFormatter::class,
            'constructor' => [
                'format' => null,
                'dateFormat' => null,
                'allowInlineLineBreaks' => true,
            ],
        ],
    ],
];
```

如果您希望再進行更細粒度的日誌切割，也可通過繼承 `Monolog\Handler\RotatingFileHandler` 類並重新實現 `rotate()` 方法實現。

### 配置多個 `Handler`

用户可以修改 `handlers` 讓對應日誌組支持多個 `handler`。比如以下配置，當用户投遞一個 `INFO` 級別以上的日誌時，只會在 `hyperf.log` 中寫入日誌。
當用户投遞一個 `DEBUG` 級別以上日誌時，會在 `hyperf.log` 和 `hyperf-debug.log` 寫入日誌。

```php
<?php

declare(strict_types=1);

use Monolog\Handler;
use Monolog\Formatter;
use Monolog\Logger;

return [
    'default' => [
        'handlers' => [
            [
                'class' => Handler\StreamHandler::class,
                'constructor' => [
                    'stream' => BASE_PATH . '/runtime/logs/hyperf.log',
                    'level' => Logger::INFO,
                ],
                'formatter' => [
                    'class' => Formatter\LineFormatter::class,
                    'constructor' => [
                        'format' => null,
                        'dateFormat' => null,
                        'allowInlineLineBreaks' => true,
                    ],
                ],
            ],
            [
                'class' => Handler\StreamHandler::class,
                'constructor' => [
                    'stream' => BASE_PATH . '/runtime/logs/hyperf-debug.log',
                    'level' => Logger::DEBUG,
                ],
                'formatter' => [
                    'class' => Formatter\JsonFormatter::class,
                    'constructor' => [
                        'batchMode' => Formatter\JsonFormatter::BATCH_MODE_JSON,
                        'appendNewline' => true,
                    ],
                ],
            ],
        ],
    ],
];

```

結果如下

```
==> runtime/logs/hyperf.log <==
[2019-11-08 11:11:35] hyperf.INFO: 5dc4dce791690 [] []

==> runtime/logs/hyperf-debug.log <==
{"message":"5dc4dce791690","context":[],"level":200,"level_name":"INFO","channel":"hyperf","datetime":{"date":"2019-11-08 11:11:35.597153","timezone_type":3,"timezone":"Asia/Shanghai"},"extra":[]}
{"message":"xxxx","context":[],"level":100,"level_name":"DEBUG","channel":"hyperf","datetime":{"date":"2019-11-08 11:11:35.597635","timezone_type":3,"timezone":"Asia/Shanghai"},"extra":[]}
```
