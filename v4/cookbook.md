# Slim 4 烹飪書

## 結尾斜線路由模式 /

Slim 將 URL 包含斜線 (/route/name/) 與不包含斜線視 (/route/name) 為不同, 他們可以使用不同的回調.

對於 GET 請求來說, 永久轉址是可以的. 但對於像 POST 或是 PUT 方法瀏覽器會發送兩次請求, 為了避免這類型況, 你只需要刪除尾部斜線並將處理後的 URL 傳遞至下一個中間層.

如果你希望包含斜線與不包含斜線的 URL 相等, 則可以添加中間層, 參考下列範例：

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->add(function (Request $request, RequestHandler $handler) {
    $uri = $request->getUri();
    $path = $uri->getPath();

    if ($path != '/' && substr($path, -1) == '/') {
      $uri = $uri->withPath(substr($path, 0, -1));

      if ($request->getMethod() == 'GET') {
        $response = new Response();
        return $response
          ->withHeader('Location', (string) $uri)
          ->withStatus(301);
      } else {
        $request = $request->withUri($uri);
      }
    }

    return $handler->handle($request);
  });

  $app->get('/show', function (Request $request, Response $response, $args) {
    $response->getBody()->write('show: GET METHOD');
    return $response;
  });

  $app->run();

```

或者, 你可以透過 middlewares/trailing-slash 中間層強制幫你把所有 URL 加入斜線.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Middlewares\TrailingSlash;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->add(new TrailingSlash(true));

  $app->get('/show/', function (Request $request, Response $response, $args) {
    $response->getBody()->write('show: GET METHOD');
    return $response;
  });

  $app->run();

```

## 取得當前路由

如果需要存取應用程序當前路由, 你需要呼叫請求類別 `getAttribute` (RouteContext::fromRequest) 方法, 它是 Slim\Route 類的實例.

你可以利用 `getName()` 取得路由名稱, 或是透過 `getMethods()` 取得這個路由所支援的方法等等.

注意：如果你希望在應用的中間層內部訪問路由, 請將 `determineRouteBeforeAppMiddleware` 設置為 true, 否則 `getAttribute('route')` 與不存在的路由均會返回 NULL.

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Slim\Routing\RouteContext;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $app->add(function (Request $request, RequestHandler $handler) {

    $route = RouteContext::fromRequest($request)
      ->getRoute();
  
    if (empty($route))
      throw new NotFoundException($request, $response);

    var_dump([
      'name'      => $route->getName(),
      'groups'    => $route->getGroups(),
      'methods'   => $route->getMethods(),
      'arguments' => $route->getArguments() 
    ]);
    
    return $handler->handle($request);
  });

  $app->addRoutingMiddleware();
  $app->get('/show', function (Request $request, Response $response, $args) {
    $response->getBody()->write('show: GET METHOD');
    return $response;
  });

  $app->run();

```

## 跨域存取設置(CORS)

### **基本解決方案**

基本的 CORS 請求, 伺服器僅需要在響應時加入下列訊息：

```
Access-Control-Allow-Origin: <domain>, ... 
```

以下為參考範例

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Slim\Exception\HttpNotFoundException;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();
  $app->addRoutingMiddleware();

  $app->options('/{routes:+}', function (Request $request, Response $response, $args) {
    return $response;
  });

  $app->add(function (Request $request, RequestHandler $handler) {
    $response = $handler->handle($request);
    return $response
      ->withHeader('Access-Control-Allow-Origin', 'http://cors.internal')
      ->withHeader('Access-Control-Allow-Headers', 'X-Requested-With, Content-Type, Accept, Origin, Authorization')
      ->withHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH, OPTIONS');
  });

  $app->get('/test', function (Request $request, Response $response, $args) {
    $payload = json_encode(['Hello' => 'World'], 384);
    $response->getBody()->write($payload);
    return $response->withHeader('Content-Type', 'application/json');
  });

  $app->map(['GET', 'POST', 'PUT', 'DELETE', 'PATCH'], '/{routes:.+}', function (Request $request, Response $response) {
      throw new HttpNotFoundException($request);
  });

  $errorMiddleware = $app->addErrorMiddleware(true, true, true);
  $errorHandler = $errorMiddleware->getDefaultErrorHandler();
  $errorHandler->forceContentType('application/json');

  $app->run();

```

Access-Control-Allow-Methods

以下中間層可用於查詢 Slim 的路由器，並取得被實作的特定方法模式. 參考下列範例：

```
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;
  use Slim\Routing\RouteContext;
  use Slim\Routing\RouteCollectorProxy;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  // Access-Control-Allow-Methods 表頭加入允許的方法
  $app->add(function (Request $request, RequestHandler $handler) {
    $allowedMethods = RouteContext::fromRequest($request)
      ->getRoutingResults()
      ->getAllowedMethods();

    $response = $handler->handle($request);
    $response = $response->withHeader(
      'Access-Control-Allow-Methods',
      implode(',', $allowedMethods)
    );

    return $response;
  });

  // 在 addRoutingMiddleware 之後加入路由
  $app->addRoutingMiddleware();
  
  $app->get('/api/{id}', function (Request $request, Response $response, $args) {
    $response->getBody()->write('GET');
    return $response;
  });

  $app->post('/api/{id}', function (Request $request, Response $response, $args) {
    $response->getBody()->write('POST');
    return $response;
  });

  $app->patch('/api/{id}', function (Request $request, Response $response, $args) {
    $response->getBody()->write('PATCH');
    return $response;
  });

  $app->delete('/api/{id}', function (Request $request, Response $response, $args) {
    $response->getBody()->write('DELETE');
    return $response;
  });

  $app->group('/api', function (RouteCollectorProxy $group) {
    $group->map(
      ['PUT', 'OPTIONS'],
      '/{user_id:[0-9]+}',
      function (Request $request, Response $response, $args) {
        $method = $request->getMethod();
        $response->getBody()->write($method);
        return $response;
      });
  });

  $app->run();

```

## 透過 POST 表單上傳檔案

透過 POST 表單上傳時, 可以使用請求的 getUploadedFiles() 方法; 

- 記得確定你使用的 form 標籤 enctype="multipart/form-data", 否則 getUploadedFiles() 只會返回空陣列. 
- 如果有多組檔案上傳並使用相同的 input name 時記得添加陣列括號(方括號), 否則 getUploadedFiles() 只會返回一個上傳的文件.

下面是一個範例 HTML 表單, 包含單組上傳與多組上傳

```
# templates/upload.html
<!DOCTYPE html>
<html>
  <head>
    <title>UPLOAD</title>
  </head>
  <body>
    <h2>UPLOAD</h2>
    <form action="/upload" method="post" enctype="multipart/form-data">
    <p>
      <label>Add file (single):</label><br />
      <input type="file" name="example1" />
    </p>
    <p></p>
    <p>
      <label>Add files (max2):</label><br />
      <input type="file" name="example2[]" /><br />
      <input type="file" name="example2[]" />
    </p>
    <p></p>
    <p>
      <label>Add files (multiple):</label><br />
      <input type="file" name="example3[]" multiple="multiple" />
    </p>
  
    <p>
      <input type="submit" />
    </p>
  </form>
</body>
</html>
```

```
# index.php
<?php

  use DI\Container;
  use Psr\Http\Message\UploadedFileInterface;
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Container\ContainerInterface as ContainerInterface;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';

  $container = new Container();
  $container->set('upload_dir', __DIR__ . '/../uploads');
  AppFactory::setContainer($container);
  $app = AppFactory::create();

  $container->set('view', function (ContainerInterface $c) {
    return new \Slim\Views\Twig(
      'templates', 
      ['cache' => 'cache']
    );
  });

  // 上傳頁面
  $app->get('/index', function (Request $request, Response $response, $args) {
    $view = $this->get('view');
    return $view
      ->render($response, 'upload.html');
  });

  // 執行上傳
  $app->post('/upload', function (Request $request, Response $response) {
    $directory = $this->get('upload_dir');
    $uploadedFiles = $request->getUploadedFiles();

    $uploadedFile = $uploadedFiles['example1'];
    if ($uploadedFile->getError() === UPLOAD_ERR_OK) {
      $filename = moveUploadedFile($directory, $uploadedFile);
      $response->getBody()->write('uploaded ' . $filename . '<br/>');
    }

    foreach ($uploadedFiles['example2'] as $uploadedFile) {
      if ($uploadedFile->getError() === UPLOAD_ERR_OK) {
        $filename = moveUploadedFile($directory, $uploadedFile);
        $response->getBody()->write('uploaded ' . $filename . '<br/>');
      }
    }

    foreach ($uploadedFiles['example3'] as $uploadedFile) {
      if ($uploadedFile->getError() === UPLOAD_ERR_OK) {
        $filename = moveUploadedFile($directory, $uploadedFile);
        $response->getBody()->write('uploaded ' . $filename . '<br/>');
      }
    }

    return $response;
  });

  function moveUploadedFile($directory, UploadedFileInterface $uploadedFile)
  {
    $filename = sprintf(
      '%s.%0.8s',
      bin2hex(random_bytes(8)),
      pathinfo(
        $uploadedFile->getClientFilename(), 
        PATHINFO_EXTENSION
      )
    );

    $uploadedFile->moveTo($directory . DIRECTORY_SEPARATOR . $filename);
    return $filename;
  }

  $app->run();
  
```