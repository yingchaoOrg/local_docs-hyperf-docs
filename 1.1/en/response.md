# 响应

在 Hyperf 里可通过 `Hyperf\HttpServer\Contract\ResponseInterface` 接口类来注入 `Response` 代理对象对响应进行处理，默认返回 `Hyperf\HttpServer\Response` 对象，该对象可直接调用所有 `Psr\Http\Message\ResponseInterface` 的方法。

## 返回 Json 格式

`Hyperf\HttpServer\Contract\ResponseInterface` 提供了 `json($data)` 方法用于快速返回 `Json` 格式，并设置 `Content-Type` 为 `application/json`，`$data` 接受一个数组或为一个实现了 `Hyperf\Utils\Contracts\Arrayable` 接口的对象。

```php
<?php
namespace App\Controller;

use Hyperf\HttpServer\Contract\ResponseInterface;
use Psr\Http\Message\ResponseInterface as Psr7ResponseInterface;

class IndexController
{
    public function json(ResponseInterface $response): Psr7ResponseInterface
    {
        $data = [
            'key' => 'value'
        ];
        return $response->json($data);
    }
}
```

## 返回 Xml 格式

`Hyperf\HttpServer\Contract\ResponseInterface` 提供了 `xml($data)` 方法用于快速返回 `XML` 格式，并设置 `Content-Type` 为 `application/xml`，`$data` 接受一个数组或为一个实现了 `Hyperf\Utils\Contracts\Xmlable` 接口的对象。

```php
<?php
namespace App\Controller;

use Hyperf\HttpServer\Contract\ResponseInterface;
use Psr\Http\Message\ResponseInterface as Psr7ResponseInterface;

class IndexController
{
    public function xml(ResponseInterface $response): Psr7ResponseInterface
    {
        $data = [
            'key' => 'value'
        ];
        return $response->xml($data);
    }
}
```

## 返回 Raw 格式

`Hyperf\HttpServer\Contract\ResponseInterface` 提供了 `raw($data)` 方法用于快速返回 `raw` 格式，并设置 `Content-Type` 为 `plain/text`，`$data` 接受一个字符串或为一个实现了 `__toString()` 方法的对象。

```php
<?php
namespace App\Controller;

use Hyperf\HttpServer\Contract\ResponseInterface;
use Psr\Http\Message\ResponseInterface as Psr7ResponseInterface;

class IndexController
{
    public function raw(ResponseInterface $response): Psr7ResponseInterface
    {
        return $response->raw('Hello Hyperf.');
    }
}
```

## 返回视图

Hyperf 暂不支持视图返回，欢迎社区贡献相关的 PR。

## 重定向

`Hyperf\HttpServer\Contract\ResponseInterface` 提供了 `redirect(string $toUrl, int $status = 302, string $schema = 'http')`  返回一个已设置重定向状态的 `Psr7ResponseInterface` 对象。

`redirect` 方法：   

|        参数          | 类型   |      默认值      |        备注        |
|:-------------------:|:------:|:---------------:|:------------------:|
|        toUrl        | string |       无        |     如果参数不存在 `http://` 或 `https://` 则根据当前服务的 Host 自动拼接对应的 URL，且根据 `$schema` 参数拼接协议     |
|        status       | int    |       302       |     响应状态码     |
|        schema       | string |       http      |    当 `$toUrl` 不存在 `http://` 或 `https://` 时生效，仅可传递 `http` 或 `https`    |

```php
<?php
namespace App\Controller;

use Hyperf\HttpServer\Contract\ResponseInterface;
use Psr\Http\Message\ResponseInterface as Psr7ResponseInterface;

class IndexController
{
    public function redirect(ResponseInterface $response): Psr7ResponseInterface
    {
        // redirect() 方法返回的是一个 Psr\Http\Message\ResponseInterface 对象，需再 return 回去  
        return $response->redirect('/anotherUrl');
    }
}
```

## Cookie 设置

```php
<?php
namespace App\Controller;

use Hyperf\HttpServer\Contract\ResponseInterface;
use Psr\Http\Message\ResponseInterface as Psr7ResponseInterface;
use Swoft\Http\Message\Cookie\Cookie;

class IndexController
{
    public function cookie(ResponseInterface $response): Psr7ResponseInterface
    {
        $cookie = new Cookie('key', 'value');
        return $response->withCookie($cookie)->withContent('Hello Hyperf.');
    }
}
```

## Gzip 压缩

## 分块传输编码 Chunk

## 返回文件下载
