# Slim 4 路由

## 概觀 (Overview)

Slim 框架路由預設 Fast Route 套件作為組件, 它非常地快速與穩定. 當我們使用這個組件來作為我們的路由時, Slim 核心與 Fast Route 套件是分離的, 並做好 Interface 來使用其他的路由庫.

## 如何創建路由

定義好應用路由並使用代理方法在 Slim\App 實例上. Slim Framework 提供豐富的 HTTP 存取方法.

- GET 方法路由

你可以使用 Slim 應用程序的 get() 方法添加處理 GET 的 HTTP 請求路由. 它接以下兩組參數.

  1. 路由表達式, 它允許選用名稱.
  2. 路由回調.

```
$app->get('/books/{id}', function ($request, $response, $args) {
  // ...
});
```

- POST 方法路由

你可以使用 Slim 應用程序的 post() 方法添加處理 POST 的 HTTP 請求路由. 它接以下兩組參數.

  1. 路由表達式, 它允許選用名稱.
  2. 路由回調.

```
$app->post('/books', function ($request, $response, $args) {
  // ...
});
```

- PUT 方法路由

你可以使用 Slim 應用程序的 put() 方法添加處理 PUT 的 HTTP 請求路由. 它接以下兩組參數.

  1. 路由表達式, 它允許選用名稱.
  2. 路由回調.

```
$app->put('/books/{id}', function ($request, $response, $args) {
  // ...
});
```
- DELETE 方法路由

你可以使用 Slim 應用程序的 delete() 方法添加處理 DELETE 的 HTTP 請求路由. 它接以下兩組參數.

  1. 路由表達式, 它允許選用名稱.
  2. 路由回調.

```
$app->delete('/books/{id}', function ($request, $response, $args) {
  // ...
});
```

- OPTIONS 路由

你可以使用 Slim 應用程序的 options() 方法添加處理 OPTIONS 的 HTTP 請求路由. 它接以下兩組參數.

  1. 路由表達式, 它允許選用名稱.
  2. 路由回調.

```
$app->options('/books/{id}', function ($request, $response, $args) {
  // ...
});
```

- PATCH 路由

你可以使用 Slim 應用程序的 patch() 方法添加處理 PATCH 的 HTTP 請求路由. 它接以下兩組參數.

  1. 路由表達式, 它允許選用名稱.
  2. 路由回調.

```
$app->patch('/books/{id}', function ($request, $response, $args) {
  // ...
});
```

- 任意路由

你可以使用 Slim 應用程序的 any() 方法添加處理所有的 HTTP 請求路由. 它接以下兩組參數.

  1. 路由表達式, 它允許選用名稱.
  2. 路由回調.

```
$app->any('/books/[{id}]', function ($request, $response, $args) {
  // ...
});
```

注意第二個參數是回調. 你可以建立一個類並使用 __invoke() 方法來替代閉包. 並且, 你可以在其他位置執行映射.

```
$app->any('/user', 'MyRestfulController');
```

- 路由組合

你可以使用 Slim 應用程序的 map() 方法添加處理多組特定的 HTTP 請求路由. 它接以下三組參數.

  1. 路由方法.
  2. 路由表達式, 它允許選用名稱.
  3. 路由回調.

```
$app->map(['GET', 'POST'], '/books', function ($request, $response, $args) {
  // ...
});
```

## 路由回調

前述所有路由方法都接受回調任務作為其最終參數. 這個變數可以被任何 PHP 所調用, 並預設是接收以下三組參數.

- Request

第一個參數為 Psr\Http\Message\ServerRequestInterface 物件, 做為表示 HTTP 請求.

- Response

第二個參數為 Psr\Http\Message\ResponseInterface 物件, 做為表示當前的 HTTP 響應.

- Arguments

第三個參數為陣列, 它包含當前路由被命名的保留變數.

### **將內容寫入響應**

你可以透過兩種方式寫入內容至 HTTP 響應:

- 透過 echo()
- 透過回傳 Psr\Http\Message\ResponseInterface 物件

### **閉包綁定**

如果你使用依賴容器(dependency container)與閉包作為路由回調, 閉包將會被綁定在容器的實例上. 這也意味著你可以透過 $this 關鍵字訪問閉包內容器實例.

```
$app->get('/hello/{name}', function ($request, $response, $args) {
  $this->get('cookies')->set('name', [
    'value' => $args['name'],
    'expires' => '7 days'
  ]);
});
```

```
注意：
Slim 不支援靜態閉包
```

## 重導向輔助

你可以使用 Slim 應用程序的 redirect() 方法添加 HTTP 請求重導向至其他路由. 它接受下列三組參數.

- 當前的路由
- 重導向的目標路由, 也可以 Psr\Http\Message\UriInterface
- 導向的 HTTP 狀態碼; 選用, 預設為 302.

```
$app->redirect('/books', '/library', 301);
```

## 路由策略(Route strategies)

路由回調簽名(route callback signature)由路由策略決定. Slim 預期接受回調的參數為請求, 響應與當前路由被命名的保留變數陣列. 這被稱呼為 RequestResponse 策略. 然而, 你可以簡單地改用不同的策略來替代路由回調簽名. 例如, Slim 提供一個允許請求, 響應加上當前路由被命名的保留變數陣列均獨立, 名為 RequestResponseArgs 的替代策略.

```
<?php
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Handlers\Strategies\RequestResponseArgs;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $routeCollector = $app->getRouteCollector();
  $routeCollector->setDefaultInvocationStrategy(new RequestResponseArgs());

  $app->get('/hello/{name}', function (Request $request, Response $response, $name) {
    return $response->write($name);
  });

  $app->run();
```

另外, 你可以基於每個路由設置不同調用策略:

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Handlers\Strategies\RequestResponseArgs;
  
  require __DIR__ . '/../vendor/autoload.php';
  
  $app = AppFactory::create();
  $routeCollector = $app->getRouteCollector();

  $route_a = $app->get('/hello/{name}', function (Request $request, Response $response, $name) {
    $response->getBody()->write('Hello, ' . $name);
    return $response;
  });
  
  $route_a->setInvocationStrategy(new RequestResponseArgs());

  $route_b = $app->get('/goodbye/{name}', function (Request $request, Response $response, $name) {
    $response->getBody()->write('Goodbye, ' . $name);
    return $response;
  });
  
  $route_b->setInvocationStrategy(new RequestResponseArgs());

  $app->run();
```

你可以透過 Slim\Interface\InvocationStrategyInterface  自行實作路由策略.

## 路由佔位符(Route placeholders)

上述每種路由均可利用當前 HTTP 請求的 URI 利用正規化方式配對. 路由會利用對應佔位符轉為變數, 並去動態匹配符合規則的路由 URI 段.

- 標準佔位符<br>
  路由將 `{變數名稱}` 視為佔位符, 範例如下:

```
$app->get('/hello/{name}', function (Request $request, Response $response, $args) {
  $name = $args['name'];
  echo "Hello, $name";
});
```

- 可選佔位符<br>
  路由用 `[/{變數名稱}]` 視為可選佔位符, 範例如下:

```
$app->get('/users[/{id}]', function ($request, $response, $args) {
  // ... 可以是 /users, /user/123
});
```

- 多組可選占位符<br>
  路由用 `[/{變數名稱}]` 視為可選佔位符, 占位符有順序性.

```
$app->get('/news[/{year}][/{month}]', function ($request, $response, $args) {
  // ... 可以是 /news, /news/2018, /news/2018/09
});
```

- 無限多組可選占位符<br>
  路由用 `[/{變數名稱}]` 視為可選佔位符, 占位符需自行由字串轉換成陣列.

```
$app->get('/news[/{params:.*}]', function ($request, $response, $args) {
  // ... 可以是 /news, /news/2018, /news/2018/09, /news/2018/09/11
  // 下面將轉換成 ['2018', '09', '11']
  $params = explode('/', $args['params']);
});
```

- 正規表示路由<br>

預設情況的占位符是寫在{ }(花括符)內, 可以接受任何值. 佔位符也可以利用於 HTTP 的請求 URI 時以正規標示法進行配對. 如果當前的 HTTP 請求不符合該正規表示法配對, 該路由不會被觸發. 下列是一個以 id 作為命名, 並要求符合一組以上的數字規則.

```
$app->get('/users/{id:[0-9]+}', function ($request, $response, $args) {
  // Find user identified by $args['id']
});
```

## 路由名稱

可以為應用程序路由指派名稱. 如果你希望透過路由解析器 urlFor() 方法產生一組特定路由時, 這很有用. 任意路由方法會返回 Slim\Route 物件, 並且該物件會有一組公開方法 setName().

```
$app->get('/hello/{name}', function ($request, $response, $args) {
  echo "Hello, " . $args['name'];
})->setName('hello');
```

你可以透過路由解析器 urlFor() 方法產生一組對應路由

```
$routeParser = $app->getRouteCollector()->getRouteParser();
echo $routeParser->urlFor('hello', ['name' => 'Josh'], ['example' => 'name']);

// Outputs "/hello/Josh?example=name"
```

路由解析器 urlFor() 接受以下三組方法:

  - $routeName 路由名稱: 透過 $route->setName('name') 去設置. 路由匹配方法會返回一個實例, 因此你可以直接設置名稱.

```
$app->get('/', function () {
  // ...
})->setName('name');
```
  - $data 路由佔位符陣列: 找到匹配的路由後, 會根據此陣列的 key-value 值去替代占位符變數值.
  - $queryParams 查詢參數: 找到匹配的路由後, 會根據此陣列的 key-value 值去組成查詢參數, 並附加於 url 上.

## 路由群組

為了幫助組織路由成為有邏輯性群集, Slim\App 提供了 group() 方法. 每個群組的路由都預先包含了相同的路由模式, 而且任何在群集的占位符變數最終都可以被做成巢狀路由.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Routing\RouteCollectorProxy;
  
  require __DIR__ . '/../vendor/autoload.php';
  
  $app = AppFactory::create();
  $app->group('/users/{id:[0-9]+}', function (RouteCollectorProxy $group) {

    $group->map(
      ['GET', 'POST', 'PATCH', 'PUT'],
      '',
      function (Request $request, Response $response, $args) {
        $response->getBody()->write('URI: /users/' . $args['id']);
        return $response;
      }
    );

    $group->get('/reset-password', function (Request $request, Response $response, $args) {
      $response->getBody()->write('URI: /users/' . $args['id'] . '/reset-password');
      return $response;
    });

  });

  $app->run();

```

群組路由模式可以為空, 授予邏輯群集不分享共通的路由模式.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Routing\RouteCollectorProxy;
  
  require __DIR__ . '/../vendor/autoload.php';
  
  $app = AppFactory::create();
  $app->group('', function (RouteCollectorProxy $group) {

    $group->get('/billing', function (Request $request, Response $response, $args) {
        $response->getBody()->write('URI: /billing');
        return $response;
    });

    $group->get('/invoice/{id:[0-9]+}', function (Request $request, Response $response, $args) {
        $response->getBody()->write('URI: /invoice/' . $args['id']);
        return $response;
    });

  });

  $app->run();

```

注意在群組閉包內部, <br>
Slim 將路由閉包內 $this 是綁定在 Psr\Container\ContainerInterface 上的容器實例.

## 路由中間層

你同樣可以將中間層掛在任何路由或路由群組上.

```
# GroupMiddleware.php
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Psr7\Response;

  class GroupMiddleware
  {
    public function __invoke(Request $request, RequestHandler $handler): Response
    {
      $response = $handler->handle($request);
      $existContent = (string)$response->getBody();

      $response = new Response();
      $response->getBody()->write('[Middleware:GroupMiddleware]' . $existContent);
      
      return $response;
    }
  }

# RouteMiddleware.php
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Psr7\Response;

  class RouteMiddleware
  {
    public function __invoke(Request $request, RequestHandler $handler): Response
    {
      $response = $handler->handle($request);
      $existContent = (string)$response->getBody();

      $response = new Response();
      $response->getBody()->write('[Middleware:RouteMiddleware]' . $existContent);
      
      return $response;
    }
  }

# index.php
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Routing\RouteCollectorProxy;
  
  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/GroupMiddleware.php';
  require __DIR__ . '/RouteMiddleware.php';
  
  $app = AppFactory::create();
  $app->group('/foo', function (RouteCollectorProxy $group) {

    $group->get('/bar', function (Request $request, Response $response, $args) {
      $response->getBody()->write('URI: /foo/bar');
      return $response;
    })->add(new RouteMiddleware());

  })->add(new GroupMiddleware());

  $app->run();

```

## 路由表示法快取 (Route expressions caching)

可以透過 RouteCollector::setCacheFile() 方法啟動路由快取. 參考下列範例：

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  
  require __DIR__ . '/../vendor/autoload.php';
  
  $app = AppFactory::create();
  $app->get('/foo/{id:[0-9]+}', function (Request $request, Response $response, $args) {
    $response->getBody()->write('URI: /foo/' . $args['id']);
    return $response;
  });

  $app->get('/bar/{id:[0-9]+}', function (Request $request, Response $response, $args) {
    $response->getBody()->write('URI: /bar/' . $args['id']);
    return $response;
  });

  $routeCollector = $app->getRouteCollector();
  $routeCollector->setCacheFile('cache/cache.file');
  $app->run();
  
```

## 容器解析 (Container Resolution)

你不僅限於為路由定義功能. 在 Slim 中, 有幾種不同的方式可以為你的路由定義操作功能.

除功能外, 你還可以使用：
- 容器:方法
- 類:方法
- 類, 並實作 __invoke() 方法
- 容器名稱

Slim 回調解析類啟動了這功能：他可以轉換將字串轉換為可回調的方法：

```
$app->get('/', '\HomeController:home');
```

另外, 你可以利用 PHP 的 ::class 運算符, 該運算符可以與上述的動作相同.

```
$app->get('/', \HomeController::class . ':home');
```

#### 這組範例程式碼是執行 HomeController 類的 home 方法.

Slim 第一步會搜尋 HomeController 容器的列表. 找到後會使用該實例並且會回調容器的建構子當作第一個參數. 一旦建立該類的實例後, 它將使用你定義的任何策略指定方法.

### **於容器註冊一組控制器**

註冊一組控制器並帶有 home 操作方法. 這組建構子會同意必須的依賴, 參考範例：

```
# HomeController.php
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;

  class HomeController
  {
    protected $view;
    public function __construct($view)
    {
      $this->view = $view;
    }

    public function home(Request $request, Response $response, $args)
    {
      $response->getBody()->write('route: /home');
      return $response;
    }
  }

# index.php
<?php

  use DI\Container;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Container\ContainerInterface as ContainerInterface;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/HomeController.php';
  $container = new Container();

  AppFactory::setContainer($container);
  $app = AppFactory::create();

  $container->set('view', function (ContainerInterface $c) {
    return new \Slim\Views\Twig(
      'templates', 
      ['cache' => 'cache']
    );
  });

  $container->set('HomeController', function (ContainerInterface $c) {
    $view = $c->get('view');
    return new HomeController($view);
  });

  $app->get('/home', 'HomeController:home');
  $app->run();

```

這使你可以利用容器進行依賴注入, 並且你可以將特定依賴注入控制器中.

### **Slim實例化控制器**

另外, 如果類別的容器沒有入口, Slim 會傳送容器至實例的建構子. 你可以建構內含多種方法的控制器.

```
# HomeController.php
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Container\ContainerInterface as ContainerInterface;

  class HomeController
  {
    protected $container;
    public function __construct(ContainerInterface $container)
    {
      $this->container = $container;
    }

    public function home(Request $request, Response $response, $args)
    {
      $response->getBody()->write('route: /home');
      return $response;
    }

    public function contact(Request $request, Response $response, $args)
    {
      $response->getBody()->write('route: /contact');
      return $response;
    }
  }

# index.php
<?php

  use DI\Container;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Container\ContainerInterface as ContainerInterface;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/HomeController.php';
  $container = new Container();

  AppFactory::setContainer($container);
  $app = AppFactory::create();

  $container->set('HomeController', function (ContainerInterface $c) {
    return new HomeController($c);
  });

  $app->get('/home', 'HomeController:home');
  $app->get('/contact', 'HomeController:contact');
  $app->run();

```

### **使用類的魔術方法**

你可以直接將類設置為可調用的類, 參考範例：

```
# HomeAction.php
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Container\ContainerInterface as ContainerInterface;

  class HomeAction
  {
    protected $container;
    public function __construct(ContainerInterface $container)
    {
      $this->container = $container;
    }

    public function __invoke(Request $request, Response $response, $args)
    {
      $response->getBody()->write('route: /home');
      return $response;
    }
  }

# index.php
<?php

  use DI\Container;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Container\ContainerInterface as ContainerInterface;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/HomeAction.php';
  $container = new Container();

  AppFactory::setContainer($container);
  $app = AppFactory::create();

  $container->set('HomeAction', function (ContainerInterface $c) {
    return new HomeAction($c);
  });

  $app->get('/home', 'HomeAction');
  $app->run();

```
