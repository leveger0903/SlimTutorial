# Slim 4 基礎

## 安裝套件

````
php composer.phar create-project slim/slim-skeleton {my-app}
````

## 執行專案

````
php -S localhost:8888 -t public public/index.php
````

## 基本安裝

````
composer require slim/slim:4.0.0-beta
````

## 挑選一組 PSR-7 的請求產生器

- Slim PSR-7
- Nyholm PSR-7 | Nyhole PSR-7 Server
- Guzzle PSR-7 | Guzzle HTTP Factory
- Zend Diactoros

## 範本

````
<?php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Slim\Factory\AppFactory;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

$app->get('/', function (Request $request, Response $response, $args) {
    $response->getBody()->write("Hello world!");
    return $response;
});

$app->run();
````

## 升級到 Slim v4

首先, 你需要將 PHP 版本升級至 7.1 或更高

## v3 與 v4 版本差異

- 移除部分設置
  - addContentLengthHeader: 參考中間層章節 - Content Length Middleware
  - determineRouteBeforeAppMiddleware: 參考中間層章節 - Routing Middleware
  - outputBuffering: 參考中間層章節 - Output Buffering Middleware
  - displayErrorDetails: 參考中間層章節 - Error Handling Middleware

- 不再支援 Container
  - 在 v3 的 App::__call() 方法已移除, 因此無法再透過 $app->key_name() 呼叫.

- 路由組件的調整
  - 在 v3 的路由組件已被拆成 RouterCollector, RouterParser 和 RouterResolver, 它們有各自的接口, 你可以實作接口並透過 App 建構式注入它們.

- 全新的中間層
  - 我們解藕 Slim App 的核心功能並實作成中間層, 開發者能彈性將核心組件替換成自定義的實作.

- 中間層的執行
  - 仍然維持 LIFO 方法, 我們取消了 ermineRouteBeforeAppMiddleware 設置.

- 全新的 App Factory (APP 工廠)
  - 引入 App Factory 組件是為了減少 PSR-7 的實作與 App 核心解藕後引起的摩擦. 他會偵測透過 App 核心實作的 PSR-7 的請求產生器實例化(AppFactory::create())並啟用(AppFactory::run())而無須像 v3 傳入請求對象.
  - 可用的 PSR-7 請求產生器
    - Slim PSR-7
    - Nyholm PSR-7 | Nyhole PSR-7 Server
    - Guzzle PSR-7 | Guzzle HTTP Factory
    - Zend Diactoros

- 全新的 Routing Middleware (路由中間層)
  - 我們使用 FastRoute 滿足路由需求, 為了要實例化路由中間層, 你需要傳遞一個App::getRouteResolver()至路由中間層. 如果之前使用 determineRouteBeforeAppMiddleware 選項, 在 run() 之前添加 Middleware\RoutingMiddleware 中間層來維持 v3 版本的行為.

````
<?php
use Slim\Factory\AppFactory;
use Slim\Middleware\RoutingMiddleware;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

$routeResolver = $app->getRouteResolver();
$routingMiddleware = new RoutingMiddleware($routeResolver);
$app->add($routingMiddleware);

...

$app->run();
````

- 全新的 Error Handling Middleware (錯誤處理中間層)
  - 錯誤處理已經被實作中間層. 

````
<?php
use Slim\Factory\AppFactory;
use Slim\Middleware\ErrorMiddleware;
use Slim\Middleware\RoutingMiddleware;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

/*
 * The routing middleware should be added earlier than the ErrorMiddleware
 * Otherwise exceptions thrown from it will not be handled by the middleware
 */
$routeResolver = $app->getRouteResolver();
$routingMiddleware = new RoutingMiddleware($routeResolver);
$app->add($routingMiddleware);

/*
 * The constructor of `ErrorMiddleware` takes in 5 parameters
 
 * @param CallableResolverInterface $callableResolver -> CallableResolver implementation of your choice
 * @param ResponseFactoryInterface $responseFactory -> ResponseFactory implementation of your choice
 * @param bool $displayErrorDetails -> Should be set to false in production
 * @param bool $logErrors -> Parameter is passed to the default ErrorHandler
 * @param bool $logErrorDetails -> Display error details in error log
 * which can be replaced by a callable of your choice.
 
 * Note: This middleware should be added last. It will not handle any exceptions/errors
 * for middleware added after it.
 */
$callableResolver = $app->getCallableResolver();
$responseFactory = $app->getResponseFactory();
$errorMiddleware = new ErrorMiddleware($callableResolver, $responseFactory, true, true, true);
$app->add($errorMiddleware);

...

$app->run();

````

- 全新的 Dispatcher & Routing Results (調度器與路由結果)
  - 我們在 FastRoute 調度器外創造一個包裝器, 用來加入結果包裝器並訪問允許路徑的完整列表, 而不是當發生異常時才能訪問這些方法. Request 的 routeInfo 已廢棄並替換成 routingResults.

````
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Slim\Factory\AppFactory;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

$app->get('/hello/{name}', function (Request $request, Response $response) {
    $routingResults = $request->getAttribute('routingResults');

    // Get all of the route's parsed arguments e.g. ['name' => 'John']
    $routeArguments = $routingResults->getRouteArguments();
    $allowedMethods = $routingResults->getAllowedMethods();
    
    return $response;
});

...

$app->run();
````

- 覆蓋中間層的新方法
  - 如果你使用其他自定義的 header 或是 body 參數來覆蓋 http 參數, 你需要加上 Middleware\MethodOverrideMiddleware 中間層即可像以前一樣覆蓋該方法.

````
<?php
use Slim\Factory\AppFactory;
use Slim\Middleware\MethodOverridingMiddleware;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

$methodOverridingMiddleware = new MethodOverridingMiddleware();
$app->add($methodOverridingMiddleware);

...

$app->run();
````

- 全新內容長度中間層
  - 內容長度中間層將自動在響應加入 Content-Length 標頭. 這是為了取代 v3 的addContentLengthHeader 選項. 這個中間層應該被放在中間層核心, 以便最後執行.

````
<?php
use Slim\Factory\AppFactory;
use Slim\Middleware\ContentLengthMiddleware;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

$contentLengthMiddleware = new ContentLengthMiddleware();
$app->add($contentLengthMiddleware);

...

$app->run();
````

- 全新輸出緩衝中間件
  - 輸出緩衝中間件允許你在兩種緩衝模式切換: APPEND(預設), PREPEND. APPEND 在既存的響應內容上加上去; PREPEND 模式將創造新的響應表身上加入目前的響應內容. 這個中間層應該被放在中間層核心, 以便最後執行.

````
<?php
use Slim\Factory\AppFactory;
use Slim\Middleware\OutputBufferingMiddleware;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

$mode = OutputBufferingMiddleware::APPEND;
$outputBufferingMiddleware = new OutputBufferingMiddleware($mode);

...

$app->run();
````

## NGINX 設置方法

````
server {
  listen 80;
  server_name example.com;
  index index.php;
  error_log /path/to/example.error.log;
  access_log /path/to/example.access.log;
  root /path/to/public;

  location / {
    try_files $uri /index.php$is_args$args;
  }

  location ~ \.php {
    try_files $uri =404;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param SCRIPT_NAME $fastcgi_script_name;
    fastcgi_index index.php;
    fastcgi_pass 127.0.0.1:9000;
  }
}
````

## 部屬

- 部屬前關閉錯誤訊息顯示

````
<?php
use Slim\Factory\AppFactory;
use Slim\Middleware\ErrorMiddleware;

require __DIR__ . '/../vendor/autoload.php';

$app = AppFactory::create();

...

$callableResolver = $app->getCallableResolver();
$responseFactory = $app->getResponseFactory();

/**
 * 如果使用預先打包的錯誤中間層,
 * 在實例化錯誤中間層物件第三個選項請記得改為 false
 */

$errorMiddleware = new ErrorMiddleware($callableResolver, $responseFactory, false, true, true);
$app->add($errorMiddleware);

...

$app->run();
````

- 記得關閉 php.ini 顯示錯誤設定

````
display_errors = 0
````

- 如果你可以掌握你擁有的伺服器, 你可以透過下列其中一組部屬系統來設置部屬過程.
  - Deploybot
  - Capistrano
  - 透過 Phing, Make, Ant 等.