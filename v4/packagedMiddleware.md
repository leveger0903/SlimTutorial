# Slim 4 中間層套件

## 路由中間層 (Routing Middleware)

路由已被實作為中間層. Slim 使用 FastRoute 作為預設路由. 如果你想要使用其他的路由套件, 你可以自行實作路由接口.

Slim 組件與路由庫透過下列四種介面建立為橋梁.

- 調度器介面: DispatcherInterface
- 路由收集器介面: RouteCollectorInterface
- 路由轉換器介面: RouteParserInterface
- 路由映射器介面: RouteResolverInterface

如果你使用 defineRouteBeforeAppMiddleware 參數, 則需要在 $app->run(); 之前將 Middleware\RoutingMiddleware 中間件加入應用程序中, 以維持前述之行為.

參考下列範例：

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  
  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->addRoutingMiddleware();

  $app->get('/', function (Request $request, Response $response, $args) {
    $response->getBody()->write('Hello, Slim');
    return $response;
  });

  $app->run();
  
```

## 錯誤處理中間層 (Error Handling Middleware)

每個 Slim 框架應用程序都有一個錯誤處理程序. 可以接收所有未捕獲的 PHP 異常. 這些錯誤處理同樣也些收當前的 HTTP 請求與響應物件. 錯誤處理必須準備並返回恰當的響應物件給 HTTP 客戶端.

### **基本使用**

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  
  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->addRoutingMiddleware();

  /**
   * 
   * @param bool $displayErrorDetails: 是否顯示錯誤
   * @param bool $logErrors: 參數是否傳入預設錯誤處理
   * @param bool $logErrorDetails: 是否記錄錯誤詳細
   */

  $errorMiddleware = $app->addErrorMiddleware(true, true, true);

  $app->get('/', function (Request $request, Response $response, $args) {
    $response->getBody()->write('Hello, Slim');
    return $response;
  });

  $app->run();

```

### **客製錯誤訊息處理**

你可以映射任何類型的例外作為客製錯誤處理.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  
  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->addRoutingMiddleware();

  $errorMiddleware = $app->addErrorMiddleware(true, true, true);
  $errorMiddleware->setDefaultErrorHandler(function (
    Request $request,
    Throwable $exception,
    bool $displayErrorDetails,
    bool $logErrors,
    bool $logErrorDetails
  ) use ($app) {

    $response = $app->getResponseFactory()
      ->createResponse();
    $response->getBody()
      ->write(
        json_encode(
          ['error' => $exception->getMessage()], 
          JSON_UNESCAPED_UNICODE
        )
      );

    return $response;
    
  });

  $app->get('/', function (Request $request, Response $response, $args) {
    $response->getBody()->write('Hello, Slim');
    return $response;
  });

  $app->run();

```

### **錯誤紀錄**

如果你想要自訂錯誤到 Slim 預設的錯誤處理時, 你可以透過 logError() 方法. 

```
# Handlers/MyErrorHandler.php
<?php

  namespace Handlers;

  use Slim\Handlers\ErrorHandler as ErrorHandler;

  class MyErrorHandler extends ErrorHandler
  {
    protected function logError(string $error): void
    {
      file_put_contents(
        __DIR__ . "../../../log/log.txt",
        $error,
        FILE_APPEND
      );
    }
  }

# index.php
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Handlers\MyErrorHandler;
  
  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/Handlers/MyErrorHandler.php';

  $app = AppFactory::create();
  $app->addRoutingMiddleware();

  $errorMiddleware = $app->addErrorMiddleware(false, true, true);
  $errorMiddleware->setDefaultErrorHandler(
    new MyErrorHandler(
      $app->getCallableResolver(),
      $app->getResponseFactory()
    )
  );

  $app->get('/', function (Request $request, Response $response, $args) {
    $response->getBody()->write('Hello, Slim');
    return $response;
  });

  $app->run();

```

### **錯誤處理與呈現**

錯誤處理會偵測 content-type 並在 ErrorRenderers 幫助下返回適合的資料. AbstractErrorHandler 抽象類已經被重構並延伸至 ErrorHandler. 預設下會偵測 content-type 類型並返回適合的 ErrorHandler. ErrorHandler 支援下列幾種 content-type 類型:

- application/json
- application/xml, text/xml
- text/html
- text/plain

對於任何內容類型, 你都可以註冊自己的錯誤 ErrorHandler. 它實作於 \Slim\Interfaces\ErrorRendererInterface

```
# Handlers/MyErrorHandler.php
<?php
  
  namespace Handlers;

  use Slim\Interfaces\ErrorRendererInterface;

  class MyErrorHandler implements ErrorRendererInterface
  {
    public function __invoke(\Throwable $exception, bool $displayErrorDetails): string
    {
      return 'My awesome format';
    }
  }

# index.php
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Handlers\MyErrorHandler;
  
  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/Handlers/MyErrorHandler.php';

  $app = AppFactory::create();
  $app->addRoutingMiddleware();

  $errorMiddleware = $app->addErrorMiddleware(true, true, true);
  $errorHandler = $errorMiddleware->getDefaultErrorHandler();
  $errorHandler->registerErrorRenderer('text/html', MyErrorHandler::class);

  $app->get('/', function (Request $request, Response $response, $args) {
    $response->getBody()->write('Hello, Slim');
    return $response;
  });

  $app->run();

```

### **強制以指定 content-type 呈現**

預設來說, ErrorHandler 會透過請求中 Accept 表頭返回適當類型內容, 如果發生錯誤時你需要強制 ErrorHandler 返回指定 content-type, 參考範例:

```
$errorHandler->forceContentType('application/json');
```
### **全新的 HTTP 異常**

我們加入 HTTP 異常在應用程序中. 當原生 HTML 渲染器被調用時會返回多組包含 description 與 title 屬性供查看.

基礎類別 HttpSpecializedException 繼承 Exception 類, 並包含下列子類：

- HttpBadRequestException(400)
- HttpForbiddenException(403)
- HttpInternalServerErrorException(500)
- HttpNotAllowedException(405)
- HttpNotFoundException(404)
- HttpNotImplementedException(501)
- HttpUnauthorizedException(401)

你可以繼承 HttpSpecializedException 進行新增其他需要的響應碼. 如 504 gateway timeout 異常, 參考下列範例：

```
# <研究中>
class HttpForbiddenException extends HttpException
{
    protected $code = 504;
    protected $message = 'Gateway Timeout.';
    protected $title = '504 Gateway Timeout';
    protected $description = 'Timed out before receiving response from the upstream server.';
}
```

## 方法覆寫中間層

你可以透過 `X-Http-Method-Override` 表頭或是表身(Body)參數 `_METHOD` 進行方法覆蓋原有的請求方法. 方法覆寫中間層應該被加入在路由中間層之後.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Middleware\MethodOverrideMiddleware;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->addRoutingMiddleware();

  $methodOverrideMiddleware = new MethodOverrideMiddleware();
  $app->add($methodOverrideMiddleware);

  $app->get('/show', function (Request $request, Response $response, $args) {
    $response->getBody()->write('show: GET METHOD');
    return $response;
  });

  $app->put('/show', function (Request $request, Response $response, $args) {
    $response->getBody()->write('show: PUT METHOD');
    return $response;
  });

  $app->run();

```

## 輸出緩衝中間層

你可以選擇以 APPEND 或 PREPEND 方式輸出緩衝, 其中預設為 APPEND. APPEND 以現有的響應表身(Body)添加內容. PREPEND 則為創造一個新的響應主體後, 並添加於響應表身之前. 輸出緩衝中間層應放在中間層堆疊核心, 使它被最後執行.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Middleware\OutputBufferingMiddleware;
  use Slim\Psr7\Factory\StreamFactory;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $outputBufferingMiddleware = new OutputBufferingMiddleware(
    new StreamFactory(),
    OutputBufferingMiddleware::APPEND
  );
  $app->add($outputBufferingMiddleware);

  $app->get('/show', function (Request $request, Response $response, $args) {
    $response->getBody()->write('show: GET METHOD');
    return $response;
  });

  $app->run();

```

## 內容長度中間層

當 HTTP 請求後, 內容長度中間層在響應返回會附加 `Content-Length` 表頭. 它是用來取代 Slim 3 的 `addContentLengthHeader` (Slim 4 已廢棄).
內容長度中間層放在中間層堆疊核心, 使它被最後執行.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Slim\Factory\AppFactory;
  use Slim\Middleware\ContentLengthMiddleware;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->add(new ContentLengthMiddleware());

  $app->get('/show', function (Request $request, Response $response, $args) {
    $response->getBody()->write('show: GET METHOD');
    return $response;
  });

  $app->run();

```