# Slim 3 響應

## 第一個 Request 呼叫 + 中間層

````
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

require 'vendor/autoload.php';

$app = new Slim\App;

# 中間層
$app->add(function (REQUEST $request, RESPONSE $response, callable $next) {
  return $next($request, $response);
});

# 路由
$app->get('/foo', function (REQUEST $request, RESPONSE $response) {
  return $response;
});

$app->run();
````

## 取得 HTTP-CODE

````
$status = $response->getStatusCode();
````

## 指派新的 HTTP-CODE

````
$status = $response->withStatus(302);
````

## 取得所有的 header 值

````
$headers = $response->getHeaders();
foreach ($headers as $key => $value)
  echo $key . ": " . implode(", ", $value);
````

## 取得單一的 header 值

````
$headerValueArray = $response->getHeader('Content-Type');
$headerValueString = $response->getHeaderLine('Content-Type');
````

## 偵測是否有 header 值

````
if ($response->hasHeader('Content-Type'))
  $headerValueString = $response->getHeaderLine('Content-Type');
````

## 新增 / 變更 / 移除 Response 的單一 header

````
# 新增response的header
$newResponse = $response->withAddedHeader('Allow', 'PUT');

# 變更response的header
$newResponse = $response->withHeader('Content-type', 'application/json');

# 移除response的header
$newResponse = $response->withoutHeader('Allow');
````

## 取得 Response 的 body 值

````
$body = $response->getBody();
$body->write('Hello World');

$detail = $body->getSize(); # Slim 3 舊版不能用?方法用錯?
$detail = $body->tell();
$detail = $body->eof();
$detail = $body->isSeekable();
$detail = $body->seek();
$detail = $body->rewind();
$detail = $body->isWritable();
$detail = $body->write($string);
$detail = $body->isReadable();
$detail = $body->read($length);
$detail = $body->getContents();
$detail = $body->getMetadata($key = null); 
````

## 取得檔案的內容值

* 參考 https://github.com/guzzle/psr7

## 返回 JSON

````
$data = array('name' => 'Bob', 'age' => 40);
$newResponse = $response->withJson($data);

# 返回特定的代碼
$newResponse = $response->withJson($data, 204);
````

## 返回轉址

````
# 以 301 方式轉址至 /new-url 新網址
return $response->withRedirect('/new-url', 301);
````