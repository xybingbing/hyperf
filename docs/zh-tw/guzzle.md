# Guzzle HTTP 客戶端

[hyperf/guzzle](https://github.com/hyperf/guzzle) 元件基於 Guzzle 進行協程處理，通過 Swoole HTTP 客戶端作為協程驅動替換到 Guzzle 內，以達到 HTTP 客戶端的協程化。

## 安裝

```bash
composer require hyperf/guzzle
```

## 使用

只需要該元件內的 `Hyperf\Guzzle\CoroutineHandler` 作為處理器設定到 Guzzle 客戶端內即可轉為協程化執行，為了方便建立協程的 Guzzle 物件，我們提供了一個工廠類 `Hyperf\Guzzle\ClientFactory` 來便捷的建立客戶端，程式碼示例如下：

```php
<?php 
use Hyperf\Guzzle\ClientFactory;

class Foo {
    /**
     * @var \Hyperf\Guzzle\ClientFactory
     */
    private $clientFactory;
    
    public function __construct(ClientFactory $clientFactory)
    {
        $this->clientFactory = $clientFactory;
    }
    
    public function bar()
    {
        // $options 等同於 GuzzleHttp\Client 建構函式的 $config 引數
        $options = [];
        // $client 為協程化的 GuzzleHttp\Client 物件
        $client = $this->clientFactory->create($options);
    }
}
```

### 使用 ^7.0 版本

元件對 `Guzzle` 的依賴已經由 `^6.3` 改為 `^6.3 | ^7.0`，預設情況下已經可以安裝 `^7.0` 版本，但以下元件會與 `^7.0` 衝突。

- hyperf/metric

可以主動執行以下操作，解決衝突

```
composer require promphp/prometheus_client_php
```

- overtrue/flysystem-cos

因為依賴庫依賴了 `guzzlehttp/guzzle-services`，而其不支援 `^7.0`，故暫時無法解決。

## 使用 Swoole 配置

有時候我們想直接修改 `Swoole` 配置，所以我們也提供了相關配置項，不過這項配置在 `Curl Guzzle 客戶端` 中是無法生效的，所以謹慎使用。

> 這項配置會替換原來的配置，比如以下 timeout 會被 10 替換。

```php
<?php
use GuzzleHttp\Client;
use Hyperf\Guzzle\CoroutineHandler;
use GuzzleHttp\HandlerStack;

$client = new Client([
    'base_uri' => 'http://127.0.0.1:8080',
    'handler' => HandlerStack::create(new CoroutineHandler()),
    'timeout' => 5,
    'swoole' => [
        'timeout' => 10,
        'socket_buffer_size' => 1024 * 1024 * 2,
    ],
]);

$response = $client->get('/');

```

## 連線池

Hyperf 除了實現了 `Hyperf\Guzzle\CoroutineHandler` 外，還基於 `Hyperf\Pool\SimplePool` 實現了 `Hyperf\Guzzle\PoolHandler`。

### 原因

簡單來說，主機 TCP 連線數 是有上限的，當我們併發大到超過這個上限值時，就導致請求無法正常建立。另外，TCP 連線結束後還會有一個 TIME-WAIT 階段，所以也無法實時釋放連線。這就導致了實際併發可能遠低於 TCP 上限值。所以，我們需要一個連線池來維持這個階段，儘量減少 TIME-WAIT 造成的影響，讓 TCP 連線進行復用。

### 使用

```php
<?php
use GuzzleHttp\Client;
use Hyperf\Utils\Coroutine;
use GuzzleHttp\HandlerStack;
use Hyperf\Guzzle\PoolHandler;
use Hyperf\Guzzle\RetryMiddleware;

$handler = null;
if (Coroutine::inCoroutine()) {
    $handler = make(PoolHandler::class, [
        'option' => [
            'max_connections' => 50,
        ],
    ]);
}

// 預設的重試Middleware
$retry = make(RetryMiddleware::class, [
    'retries' => 1,
    'delay' => 10,
]);

$stack = HandlerStack::create($handler);
$stack->push($retry->getMiddleware(), 'retry');

$client = make(Client::class, [
    'config' => [
        'handler' => $stack,
    ],
]);
```

另外，框架還提供了 `HandlerStackFactory` 來方便建立上述的 `$stack`。

```php
<?php
use Hyperf\Guzzle\HandlerStackFactory;
use GuzzleHttp\Client;

$factory = new HandlerStackFactory();
$stack = $factory->create();

$client = make(Client::class, [
    'config' => [
        'handler' => $stack,
    ],
]);
```

## 使用 `ClassMap` 替換 `GuzzleHttp\Client`

如果第三方元件並沒有提供可以替換 `Handler` 的介面，我們也可以通過 `ClassMap` 功能，直接替換 `Client` 來達到將客戶端協程化的目的。

> 當然，也可以使用 SWOOLE_HOOK 達到相同的目的。

程式碼示例如下：

class_map/GuzzleHttp/Client.php

```php
<?php
namespace GuzzleHttp;

use GuzzleHttp\Psr7;
use Hyperf\Guzzle\CoroutineHandler;
use Hyperf\Utils\Coroutine;

class Client implements ClientInterface
{
    // 程式碼省略其他不變的程式碼

    public function __construct(array $config = [])
    {
        $inCoroutine = Coroutine::inCoroutine();
        if (!isset($config['handler'])) {
            // 對應的 Handler 可以按需選擇 CoroutineHandler 或 PoolHandler
            $config['handler'] = HandlerStack::create($inCoroutine ? new CoroutineHandler() : null);
        } elseif ($inCoroutine && $config['handler'] instanceof HandlerStack) {
            $config['handler']->setHandler(new CoroutineHandler());
        } elseif (!is_callable($config['handler'])) {
            throw new \InvalidArgumentException('handler must be a callable');
        }

        // Convert the base_uri to a UriInterface
        if (isset($config['base_uri'])) {
            $config['base_uri'] = Psr7\uri_for($config['base_uri']);
        }

        $this->configureDefaults($config);
    }
}

```

config/autoload/annotations.php

```php
<?php

declare(strict_types=1);

use GuzzleHttp\Client;

return [
    'scan' => [
        // ...
        'class_map' => [
            Client::class => BASE_PATH . '/class_map/GuzzleHttp/Client.php',
        ],
    ],
];
```
