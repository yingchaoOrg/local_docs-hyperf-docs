# 列舉類

當您需要定義錯誤碼和錯誤資訊時，可能會使用以下方式，

```php
<?php

class ErrorCode
{
    const SERVER_ERROR = 500;
    const PARAMS_INVALID = 1000;

    public static $messages = [
        self::SERVER_ERROR => 'Server Error',
        self::PARAMS_INVALID => '引數非法'
    ];
}

$message = ErrorCode::messages[ErrorCode::SERVER_ERROR] ?? '未知錯誤';

```

但這種實現方式並不友好，每當要查詢錯誤碼與對應錯誤資訊時，都要在當前 `Class` 中搜索兩次，所以框架提供了基於註解的列舉類。

## 安裝

```
composer require hyperf/constants
```

## 使用

### 定義列舉類

透過 `gen:constant` 命令可以快速的生成一個列舉類。

```bash
php bin/hyperf.php gen:constant ErrorCode
```

```php
<?php

declare(strict_types=1);

namespace App\Constants;

use Hyperf\Constants\AbstractConstants;
use Hyperf\Constants\Annotation\Constants;

#[Constants]
class ErrorCode extends AbstractConstants
{
    /**
     * @Message("Server Error！")
     */
    const SERVER_ERROR = 500;

    /**
     * @Message("系統引數錯誤")
     */
    const SYSTEM_INVALID = 700;
}
```

使用者可以使用 `ErrorCode::getMessage(ErrorCode::SERVER_ERROR)` 來獲取對應錯誤資訊。

### 定義異常類

如果單純使用 `列舉類`，在異常處理的時候，還是不夠方便。所以我們需要自己定義異常類 `BusinessException`，當有異常進來，會根據錯誤碼主動查詢對應錯誤資訊。

```php
<?php

declare(strict_types=1);

namespace App\Exception;

use App\Constants\ErrorCode;
use Hyperf\Server\Exception\ServerException;
use Throwable;

class BusinessException extends ServerException
{
    public function __construct(int $code = 0, string $message = null, Throwable $previous = null)
    {
        if (is_null($message)) {
            $message = ErrorCode::getMessage($code);
        }

        parent::__construct($message, $code, $previous);
    }
}
```

### 丟擲異常

完成上面兩步，就可以在業務邏輯中，丟擲對應異常了。

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Constants\ErrorCode;
use App\Exception\BusinessException;

class IndexController extends AbstractController
{
    public function index()
    {
        throw new BusinessException(ErrorCode::SERVER_ERROR);
    }
}
```

### 可變引數

在使用 `ErrorCode::getMessage(ErrorCode::SERVER_ERROR)` 來獲取對應錯誤資訊時，我們也可以傳入可變引數，進行錯誤資訊組合。比如以下情況

```php
<?php

use Hyperf\Constants\AbstractConstants;
use Hyperf\Constants\Annotation\Constants;

#[Constants]
class ErrorCode extends AbstractConstants
{
    /**
     * @Message("Params %s is invalid.")
     */
    const PARAMS_INVALID = 1000;
}

$message = ErrorCode::getMessage(ErrorCode::PARAMS_INVALID, ['user_id']);

// 1.2 版本以下 可以使用以下方式，但會在 1.2 版本移除
$message = ErrorCode::getMessage(ErrorCode::PARAMS_INVALID, 'user_id');
```

### 國際化

> 該功能僅在 v1.1.13 及往後的版本上可用

要使 [hyperf/constants](https://github.com/hyperf/constants) 元件支援國際化，就必須要安裝 [hyperf/translation](https://github.com/hyperf/translation) 元件並配置好語言檔案，如下：

```
composer require hyperf/translation
```

相關配置詳見 [國際化](zh-tw/translation.md)

```php
<?php

// 國際化配置

return [
    'params.invalid' => 'Params :param is invalid.',
];

use Hyperf\Constants\AbstractConstants;
use Hyperf\Constants\Annotation\Constants;

#[Constants]
class ErrorCode extends AbstractConstants
{
    /**
     * @Message("params.invalid")
     */
    const PARAMS_INVALID = 1000;
}

$message = ErrorCode::getMessage(ErrorCode::SERVER_ERROR, ['param' => 'user_id']);
```
