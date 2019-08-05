# Slim 4 概念

## 程序的生命週期

- 實例化 (Instantiation)
  首先實例化 Slim\App 類. 這是 Slim 的程式物件, Slim 會針對每個依賴的應用注入預設的服務. 可透過 setting 陣列設置 Slim 的物件的行為偏好.

- 路由定義 (Route Definitions)
  你將定義實例後的路由方法(get, post, put, delete, patch...), 他們將被註冊於早些實例化的 Slim 物件內. 任一個路由方法會返回路由實例, 因此你可以調用路由實例加入再中間層或是為它命名.

- 程序運行器 (Application Runner)
  當你透過 run() 調用實例, 這個方法會執行下列程序:
  
  - 進入中間層堆棧
  - 運行程序
  - 離開中間層堆棧
  - 傳送 HTTP 響應

## PSR-7

Slim 支持請求與響應使用 PSR-7 介面. 它可以使用任意的 PSR-7 實作讓 Slim 更彈性. 舉例來說如 GuzzleHttp\Psr7.

Slim 也提供自己的 PSR-7 實作, 因此它可開箱即用. 你可以自由地用第三方套件替換成替換 Slim 預設的 PSR-7 物件. 只要覆寫應用程序容器的請求與響應服務, 讓他們分別回傳 Psr\Http\Message\ServerRequestInterface 與 Psr\Http\Message\ResponseInterface.

### Value objects

請求與響應是不可變值的物件. 只有當更新屬性值的複製版本, 才會以克隆方式被更改; 你可以藉由調用任何一個 PSR-7 的方法來請求一個價值物件(value object), 如 withHeader($name, $value)

```
<?php
  
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->get('/foo', function (Request $request, Response $response, array $args) {
    $payload = json_encode(['hello' => 'world'], JSON_PRETTY_PRINT);
    $response->getBody()->write($payload);
    return $response->withHeader('Content-Type', 'application/json');
  });

  $app->run();

```

以下為 PSR-7 提供轉換請求與響應物件的方法

- withProtocolVersion($version)
- withHeader($name, $value)
- withAddedHeader($name, $value)
- withoutHeader($name)
- withBody(StreamInterface $body)

以下為 PSR-7 提供轉換請求物件的方法

- withMethod($method)
- withUri(UriInterface $uri, $preserveHost = false)
- withCookieParams(array $cookies)
- withQueryParams(array $query)
- withUploadedFiles(array $uploadedFiles)
- withParsedBody($data)
- withAttribute($name, $value)
- withoutAttribute($name)

以下為 PSR-7 提供轉換響應物件的方法

- withStatus($code, $reasonPhrase = '')

## 中間層 (Middleware)

你可以在 Slim 應用程序運行前後透過中間層操作請求與響應物件. 中間件適合使用於檢查跨站請求偽造(CSRF), 請求之前的驗證等場景.

### 關於中間層

中間層是根據 PSR-15 中間層介面所實作的.

- Psr\Http\Message\ServerRequestInterface - PSR-7 請求物件
- Psr\Http\Server\RequestHandleInterface - PSR-15 請求處理物件

它可以做任何適合這些物件的事情. 但它們必須返回一個 Psr\Http\Message\ServerRequestInterface 的實例. 任一中間層應該調用下一個中間層並且將請求與響應物件當作參數.

### 運作方式

不同的框架使用不同的中間件, Slim 以核心應用程序為核心以同心圓方式添加中間層, 添加的最後一個中間件事第一個被執行的.

### 如何撰寫

中間層可調用兩個參數：請求物件與請求處理物件, 每個中間件必須返回一個 Psr\Http\Message\ResponseInterface 實例.

以下為閉包中間層範例：

```
<?php
  
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Slim\Psr7\Response;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $beforeMiddleware = function (Request $request, RequestHandler $handler) {
    $response = $handler->handle($request);
    $existingContent = (string) $response->getBody();

    $response = new Response();
    $response->getBody()->write('BEFORE' . $existingContent);

    return $response;
  };

  $afterMiddleware = function ($request, $handler) {
    $response = $handler->handle($request);
    $response->getBody()->write('AFTER');
    return $response;
  };

  $app->add($beforeMiddleware);
  $app->add($afterMiddleware);

  $app->get('/application', function (Request $request, Response $response, array $args) {
    $response->getBody()->write("{Hello}");
    return $response;
  });

  $app->run();
```

以下為調用類中間件範例：

```
## BeforeMiddleware.php
<?php
  
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Psr7\Response;

  class BeforeMiddleware
  {
    public function __invoke(Request $request, RequestHandler $handler): Response
    {
      $response = $handler->handle($request);
      $existingContent = (string) $response->getBody();
    
      $response = new Response();
      $response->getBody()->write('BEFORE' . $existingContent);
    
      return $response;
    }
  }

## AfterMiddleware.php
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

  class AfterMiddleware
  {
    public function __invoke(Request $request, RequestHandler $handler): Response
    {
      $response = $handler->handle($request);
      $response->getBody()->write('AFTER');
      return $response;
    }
  }

## index.php
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/BeforeMiddleware.php';
  require __DIR__ . '/AfterMiddleware.php';
  
  $app = AppFactory::create();
  $app->add(new BeforeMiddleware());
  $app->add(new AfterMiddleware());
  
  $app->get('/application', function (Request $request, Response $response, array $args) {
    $response->getBody()->write("{Hello}");
    return $response;
  });
  
  $app->run();
  
```

### 加入中間層的時機

你可以將中間層加入 Slim 應用程序, 其中一個 Slim 應用程序路由或是群組. 所有的中間層腳本均允許使用相同的中間層並實作成相同的中間層介面.

### Slim 應用程序加入中間層

使用中間層的方式是, 每當中間層被調用於 HTTP 請求時, Slim 應用程序實例會調用 add() 方法, 以下為閉包中間層範例：

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Slim\Psr7\Response;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $app->add(function (Request $request, RequestHandler $handler) {
    $response = $handler->handle($request);
    $existingContent = (string) $response->getBody();

    $response = new Response();
    $response->getBody()->write('BEFORE ' . $existingContent);

    return $response;
  });

  $app->add(function (Request $request, RequestHandler $handler) {
    $response = $handler->handle($request);
    $response->getBody()->write(' AFTER');
    return $response;
  });

  $app->get('/', function (Request $request, Response $response, $args) {
    $response->getBody()->write('Hello World');
	  return $response;
  });

  $app->run();

```

### 單一路由加入中間層

中間層僅被調用於單一的 HTTP 網址與方法, 使用方式為在調用任何單一路由後, 在路由結尾加上 add() 方法. 以下為範例：

```
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $routeMiddleware = function (Request $request, RequestHandler $handler) {
    $response = $handler->handle($request);
    $response->getBody()->write('World');

    return $response;
  };

  $app->get('/application', function (Request $request, Response $response, $args) {
    $response->getBody()->write('Hello ');

    return $response;
  })->add($routeMiddleware);

  $app->run();

```

## 群組路由加入中間層

中間層僅被調用於特定網址與方法, 使用方式為在調用任何群組路由後, 在路由結尾加上 add() 方法. 以下為範例：

```
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Slim\Routing\RouteCollectorProxy;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $app->group('/application', function (RouteCollectorProxy $group) {
    $group->get('/date', function (Request $request, Response $response) {
      $response->getBody()->write(date('Y-m-d H:i:s'));
      return $response;
    });
    
    $group->get('/time', function (Request $request, Response $response) {
      $response->getBody()->write((string)time());
      return $response;
    });
  })->add(function (Request $request, RequestHandler $handler) use ($app) {
    $response = $handler->handle($request);
    $dateOrTime = (string) $response->getBody();

    $response = $app->getResponseFactory()->createResponse();
    $response->getBody()->write('It is now ' . $dateOrTime . '. Enjoy!');

    return $response;
  });

  $app->run();

```

### 中間層的傳遞變數

你可以在中間層設置變數後, 在路由取得中間層所設置的變數.

```
## 設置變數
$request = $request->withAttribute('date', '2019-07-01');

## 取得變數
$date = $request->getAttribute('date');
```

參照下列範例：

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Slim\Psr7\Response;
  
  require __DIR__ . '/../vendor/autoload.php';
  
  $app = AppFactory::create();

  $middleware = function (Request $request, RequestHandler $handler) {
    $request = $request->withAttribute('date', date('Y-m-d'));
    $response = $handler->handle($request);
    $existingContent = (string) $response->getBody();

    return $response;
  };
    
  $app->add($middleware);

  $app->get('/application', function (Request $request, Response $response, array $args) {
    $date = $request->getAttribute('date');
    $response->getBody()->write('It is now ' . $date . '. Enjoy!');

    return $response;
  });
  
  $app->run();

```

## 依賴容器 (Dependency Container)

Slim 使用可選依賴容器來準備, 管理與注入依賴應用程序. 支援 PSR-11 規範的容器實作套件如 PHP-DI.

### 使用 PHP-DI

如果你需要使用依賴容器, 那麼在創建 App 之前要向AppFactory 提供容器實例.

```
## MyDiService.php
<?php

  class MyDiService
  {
    private $service;

    function __construct($setting) 
    {
      $this->service = $setting['service'];
    }

    public function getServiceName() 
    {
      return $this->service;
    }
  }

<?php

  use DI\Container;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;
  
  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/MyDiService.php';
  $container = new Container();
  
  AppFactory::setContainer($container);
  $app = AppFactory::create();

  $container->set('diService', function () {
    $settings = [
      'service' => 'php-di-service'
    ];
    return new MyDiService($settings);
  });

  $app->get('/application', function (Request $request, Response $response, $args) {
    if ($this->has('diService')) {
      $myDiService = $this->get('diService');
      $name = $myDiService->getServiceName();
      $response->getBody()->write($name);
    }
    return $response;
  });

  $app->run();

```



