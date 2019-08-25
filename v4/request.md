# Slim 4 請求

## 概觀 (Overview)

你的 Slim 應用程序陸游與中間層將獲得一個 PSR-7 請求物件, 該物件表示你的 web 伺服器收到當前的 HTTP 請求. 請求物件是由 PSR-7 ServerRequestInterface 實作, 你可以使用它來查看或操作 HTTP 請求方法, 表頭與表身.

### 如何取得請求物件

PSR-7 請求物件被注入在你的 Slim 應用程序回調的第一組參數, 參考如下：

```
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $app->get('/hello', function (Request $request, Response $response) {
    $response->getBody()->write('Hello World');
    return $response;
  });

  $app->run();

```

PSR-7 請求對象被注入到你的 Slim 應用程序中間層, 作為中間層調用參考下列範例：

```
<?php
  
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';

  $app = AppFactory::create();

  $app->add(function (ServerRequestInterface $request, RequestHandler $handler) {
    return $handler->handle($request);
  });

  // ...定義路由...

  $app->run();

```

### 請求的方法

以下為常見的請求方法

 - GET
 - POST
 - PUT
 - DELETE
 - HEAD
 - PATCH
 - OPTIONS

你也可以透過以下的方法查看當前所請求的方法

```
$method = $request->getMethod();
```

可偽造或是覆蓋 HTTP 請求方法. 舉例來說, 在傳統的瀏覽器只支援 POST 與 GET 兩種請求類型, 可以透過 POST 請求 + METHOD 包含 PUT 參數 + 請求內容類型 application / x-www-form-urlencoded 來實現.

```
POST /path HTTP/1.1
Host: example.com
Content-type: application/x-www-form-urlencoded
Content-length: 22

data=value&_METHOD=PUT
```

或是透過 X-Http-Method-Override 表頭來覆寫 HTTP 請求, 這在任何請求內容類型都可使用.

```
POST /path HTTP/1.1
Host: example.com
Content-type: application/json
Content-length: 16
X-Http-Method-Override: PUT

{"data":"value"}
```

### 請求 URI

每個 HTTP 請求都有一個 URI 供識別所請求的應用資源, HTTP 請求 URI 有以下幾個部分：

  - 協定 (http, https)
  - 主機位置 (example.com)
  - 埠口 (80, 443)
  - 路徑 (/users/1)
  - 查詢字串 (?sort=created&dir=asc)

你可以使用符合 PSR-7 規範的請求物件方法：

```
  $uri = $request->getUri();
```

PSR-7 請求物件的 URI 本身也是一個物件, 它提供下列方法來查看 HTTP 請求 URL 的部分.


  - getScheme() (http)
  - getAuthority() ([username@]localhost[:8888])*
  - getUserInfo() (username[:password])*
  - getHost() (localhost)
  - getPort() (8888)
  - getPath() (/hello)
  - getBasePath()**
  - getQuery() (user=kaoru&passwd=1234)
  - getFragment() (#anchor)*
  - getBaseUrl()**

\* 通常情況下似乎無法使用<br>
\*\* 會出現錯誤(這個要研究)

基本路由

如果 Slim 應用程序的前端控制器位於實體路徑根目錄下, 則可以使用 getBasePath 方法取得 HTTP 請求的的實體路徑(根目錄開始). 如果 Slim 應用程序安裝於實體路徑根目錄之外則回傳空字串.

使用範例

```
  $uri = $request->getUri();
  echo $url->getScheme();
```

查詢字串可用 getQueryParams(), 它會自動將查詢字串轉成物件型陣列

```
  $uri = $request->getQueryParams();
```

### 請求表頭

每個 HTTP 請求都有表頭. 這些是敘述 HTTP 的請求但無法在請求的表身所看到. Slim 的 PSR-7 請求物件提供多組方法可以去查看表頭.

- getHeaders(): 取得所有的表頭

```
  $headers = $request->getHeaders();
  foreach ($headers as $name => $values) {
    echo $name . ": " . implode(", ", $values);
  }
```

- getHeader(): 取得單一表頭, 以陣列型態返回

```
  $headerValueArray = $request->getHeader('Accept');
```

- getHeaderLine(): 取得單一表頭, 以字串型態返回

```
  $headerValueString = $request->getHeaderLine('Accept');
```

- hasHeader(): 偵測是否有此表頭

```
  if ($request->hasHeader('Accept')) {
    // ...
  }
```

### 請求表身

每個 HTTP 請求都有表身. 如果你使用 Slim 應用程序開發 JSON 或 XML, 你可以使用 PSR-7 請求物件方法 getParsedBody() 來解析並轉成物件. 注意不同的 PSR-7 套件的表身輸出可能會有所不同.

你需要實作中間層, 用於轉換不同的 PSR-7 實作輸出結果, 如下列程式碼：

```
## JsonMiddleware.php
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\MiddlewareInterface;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;

  class JsonMiddleware implements MiddlewareInterface
  {
    public function process(Request $request, RequestHandler $handler): Response
    {
      $contentType = $request->getHeaderLine('Content-Type');

      if (strstr($contentType, 'application/json')) {
        $contents = json_decode(file_get_contents('php://input'), true);
        if (json_last_error() === JSON_ERROR_NONE)
          $request = $request->withParsedBody($contents);
      }

      return $handler->handle($request);
    }
  }

## index.php
<?php
  
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/JsonMiddleware.php';

  $app = AppFactory::create();
  $app->add(new JsonMiddleware());

  $app->post('/hello', function (Request $request, Response $response) {
    $body = $request->getParsedBody();
    $response->getBody()->write(json_encode($body));

    return $response->withHeader('Content-Type', 'application/json');
  });

  $app->run();

``` 

以技術上來說, PSR-7 請求物件主體相當於由一個 Psr\Http\Message\StreamInterface 實例化的 HTTP 請求主體. 你可以使用 PSR-7 請求的 getBody() 方法獲得 HTTP 請求主體的 StreamInterface 實例. 對於傳入的 HTTP 請求大小未知或過大, getBody() 方法是更合適的.

```
$body = $request->getBody();
```

Psr\Http\Message\StreamInterface 提供以下的方法來基於 PHP 資源庫的讀取或迭代方法.

  - getSize()
  - tell()
  - eof()
  - isSeekable()
  - seek
  - rewind()
  - isWritable()
  - write($string)
  - isReadable()
  - read($length)
  - getContents(): 取得 $request 內容
  - getMetadata($key = null)

  ```
  $app->post('/hello', function (Request $request, Response $response) {
    $body = $request->getBody()->getContents();
    $response->getBody()->write($body);
    return $response;
  });
  ```

### 上傳檔案

$_FILES 也可以從請求物件 getUploadedFiles() 方法取得. 它會返回一個 key-value 的 input 陣列元素.

Psr\Http\Message\UploadedFileInterface 提供以下的方法來處理上傳檔案.

- getStream()
- moveTo($targetPath): 搬移上傳檔案
- getSize(): 取得檔案大小
- getError(): 取得是否有錯誤
- getClientFilename(): 取得客戶端原始上傳名稱
- getClientMediaType(): 取得檔案類型


```
$app->post('/hello', function (Request $request, Response $response) {
  $files = $request->getUploadedFiles()['image'];

  $newPath = 'D:\images\\';
  $newFile = $files->getClientFilename();
  $files->moveTo($newPath . $newFile);
});
```

### 請求函式 (helpers)

Slim 的 PSR-7 請求實作額外的方法, 幫助你更進一步檢查 HTTP 請求.

- XHR(ajax) 請求

透過表頭 `X-Requested-With: XMLHttpRequest` 識別是否為 ajax 請求.

```
if ($request->getHeaderLine('X-Requested-With') === 'XMLHttpRequest') {
  // ...
}
```

- 內容類型

透過 getHeaderLine() 方法取得 Content-Type 表頭值來是表請求內容類型.

```
$contentType = $request->getHeaderLine('Content-Type');
```

- 內容長度

透過 getHeaderLine() 方法取得 Content-Length 表頭值來是表請求內容長度.

```
$contentType = $request->getHeaderLine('Content-Length');
```

- 請求參數

取得單個請求參數值, 透過 getServerParams() 方法, 如取得請求主機名稱:

```
$host = $request->getServerParams()['HTTP_HOST'];
```

### 路由物件

有時在中間層下, 你需要取得你的路由參數. 在以下範例下, 我們將檢查用戶是否登入, 然後用戶是否有權觀看該影片.

(情境為 route → middleware)

```
## PermissionMiddleware.php
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Psr7\Response;
  use Slim\Routing\RouteContext;

  class PermissionMiddleware
  {
    public function __invoke(Request $request, RequestHandler $handler): Response
    {
      $courseId = RouteContext::fromRequest($request)
        ->getRoute()
        ->getArgument('id');

      if ((int)$courseId !== 10) {
        $response = new Response();
        $response->getBody()->write('Not Authorized');
        return $response;
      }

      return $handler->handle($request);
    }
  }

## index.php
<?php
  
  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/PermissionMiddleware.php';

  $app = AppFactory::create();
  
  $app
    ->get('/course/{id}', function (Request $request, Response $response, $args) {

      $courseId = $args['id'];
      $response->getBody()->write($courseId);
      return $response;
    })
    ->add(PermissionMiddleware::class);

  $app->addRoutingMiddleware();
  $app->run();

```

## 屬性

使用 PSR-7, 它可以在請求物件內注入物件或是變數值供未來進一步處理. 在你的應用程序中間層需要傳入訊息至閉包並且將其屬性加入到你的請求物件時. 如: 將設置值傳入請求物件.

(情境為 middleware → route)

```
## SessionMiddleware.php
<?php

  use Psr\Http\Message\ServerRequestInterface as Request;
  use Psr\Http\Server\RequestHandlerInterface as RequestHandler;
  use Slim\Psr7\Response;

  class SessionMiddleware
  {
    public function __invoke(Request $request, RequestHandler $handler): Response
    {
      $request = $request->withAttribute('sessionId', session_id());
      return $handler->handle($request);
    }
  }

## index.php
<?php

  session_start();

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/SessionMiddleware.php';

  $app = AppFactory::create();
  
  $app->get('/session', function (Request $request, Response $response, $args) {
    $sessionId = $request->getAttribute('sessionId');
    
    $response->getBody()->write('Your session id: ' . $sessionId);
    return $response;
  })
  ->add(SessionMiddleware::class);

  $app->run();

```