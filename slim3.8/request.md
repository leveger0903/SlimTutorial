# Slim 3 Request

## 第一個 Request 呼叫

````
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

require 'vendor/autoload.php';

$app = new Slim\App;

$app->get('/foo', function (REQUEST $request, RESPONSE $response) {
  $response->write('Hello');
  return $response;
});

$app->run();
````

## 加入中間層

````
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

require 'vendor/autoload.php';

$app = new Slim\App;

# 中間層
$app->add(function (REQUEST $request, RESPONSE $response, $next) {
  $response->write('Middleware do something. ');
  return $next($request, $response);
}); 

# 路由
$app->get('/foo', function (REQUEST $request, RESPONSE $response) {
  $response->write('Route do something.');
  return $response;
});  

$app->run();
````

## 請求方法

````
# GET方法
$app->get('/foo', function (REQUEST $request, RESPONSE $response) {
  $response->write('Route do something.');
  return $response;
});

# POST方法
$app->post('/foo', function (REQUEST $request, RESPONSE $response) {
  $response->write('Route do something.');
  return $response;
});

# PUT方法
$app->put('/foo', function (REQUEST $request, RESPONSE $response) {
  $response->write('Route do something.');
  return $response;
});

# DELETE方法
$app->delete('/foo', function (REQUEST $request, RESPONSE $response) {
  $response->write('Route do something.');
  return $response;
});

# 取得或是辨識目前的方法
$request->getMethod();   // GET.POST.PUT.DELETE
$request->isGet();       // true.false
$request->isPost();      // true.false
$request->isPut();       // true.false
$request->isDelete();    // true.false
$request->isHead();      // true.false
$request->isPatch();     // true.false
$request->isOptions();   // true.false
````

- 可以利用POST模擬帶入其他的方法(PUT.DELETE等等)\
  data=123&_METHOD=PUT

- 可以利用header改寫帶入其他方法(PUT.DELETE等等)\
  X-Http-Method-Override: PUT

## 請求 URI

````
$params = $request->getQueryParams();
$uri    = $request->getUri();

$scheme    = $uri->getScheme();
$authority = $uri->getAuthority();
$userInfo  = $uri->getUserInfo();
$host      = $uri->getHost();
$port      = $uri->getPort();
$path      = $uri->getPath();
$basePath  = $uri->getBasePath();
$query     = $uri->getQuery();
$fragment  = $uri->getFragment();
$baseurl   = $uri->getBaseUrl();
````

## 請求表頭

````
# 請求所有的表頭
$multiHeader = $request->getHeaders();
foreach ($multiHeader as $key => $value)
  echo $key . ": " . implode(", ", $value);

# 請求單一表頭
$header = $request->getHeaderLine('HTTP_USER_AGENT');
  echo $header;

# 偵測表頭是否存在
if($request->hasHeader('HTTP_USER_AGENT'))
  echo 'has header: HTTP_USER_AGENT'; 
````

## 請求主體

````
# 請求所有主體
$parsedBody = $request->getParsedBody();
echo json_encode($parsedBody);

# 請求單一主體
$parsedBody = $request->getParsedBodyParam('name', $default = null);
echo $parsedBody;

# 請求未知主體, 取得更詳細的資訊
$body = $request->getBody();

$detail = $body->getContents();
$detail = $body->getSize();
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

## 上傳檔案

````
$files = $request->getUploadedFiles();
  
# <input type='file' name='upload'>
if (!empty($files['upload']))
  $newfile = $files['upload'];

$detail = $newfile->getStream();
$detail = $newfile->moveTo($targetPath);
$detail = $newfile->getSize();
$detail = $newfile->getStream();
$detail = $newfile->getError();
$detail = $newfile->getClientMediaType();
````

## 偵測 xhr (ajax)

````
$xhr = $request->isXhr();
````

## 取得 content-type / media-types / charset

````
# <meta content="text/html; charset=utf-8" http-equiv="Content-Type" />
$contentType = $request->getContentType();

# 請求內容類型：text/html
$mediaType = $request->getMediaType();

# 請求內容字集：utf-8
$charset = $request->getContentCharset();

# 請求內容長度
$length = $request->getContentLength();

# 取得特定物件名稱
$detail = $request->getParam('get_name');            // GET
$detail = $request->getQueryParam('get_name');       // GET
$detail = $request->getParsedBodyParam('post_name'); // POST
$detail = $request->getCookieParam('cookie_name');   // COOKIE
$detail = $request->getServerParam('server_name');   // SERVER
````

## 路由對象 (Route Object)

* 請參考 https://www.slimframework.com/docs/v3/objects/request.html#route-object

## 媒體類型解析 (Media Type Parsers)

* 偵測請求類型

````
# 中間層, 當偵測Content-type為text/javascript則進行JSON解譯
$app->add(function ($request, $response, $next) {
  $request->registerMediaTypeParser(
    'text/javascript',
    function ($input) {
      return json_decode($input, true);
    }
  );

  return $next($request, $response);
});

$app->post('/foo', function (REQUEST $request, RESPONSE $response, $args) {    
  $parsedBody = $request->getParsedBody();
  $response->write($parsedBody);
  return $response;
});
````

## 攜帶屬性至路由

````
# 中間層, 攜帶屬性至路由
$app->add(function ($request, $response, $next) {
  $request = $request->withAttribute('cookie', $_COOKIE);
  return $next($request, $response);
});

$app->get('/foo', function (REQUEST $request, RESPONSE $response, $args) {
  $cookie = $request->getAttribute('cookie');
  return $response->write('Yay, ' . $cookie['name']);
});
````