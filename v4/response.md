# Slim 4 響應

## 概觀 (Overview)

你的 Slim 應用程序路由與中間層會獲得一個 PSR-7 的響應物件, 它代表正要返回 HTTP 響應給客戶端. 這個響應物件是由 PSR-7 ResponseInterface 所實作, 你可以用它來查看或操作 HTTP 的響應狀態, 表頭與表身.

## 如何取得響應物件

將 PSR-7 響應物件注入在你的 Slim 應用程序路由回調的第二組參數, 參考如下：

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

無法對原始 $response 進行操作, <br>
因此需要變更 $response 內容值時需複製 $response 再進行其他操作.<br>
(僅讀取不需要)

```
$app->get('/', function (Request $request, Response $response, array $args) {

  $newResponse = $response->withHeader('allow', 'post')
    ->withAddedHeader('allow', 'put')
    ->withAddedHeader('allow', 'patch');

  $newResponse->getBody()->write('');
  return $newResponse;
});
```

## 響應狀態

每個 HTTP 響應狀態都有一個對應數字代碼. 它代表現在的 HTTP 結果並會返回給客戶端. PSR-7  的預設狀態碼是 200(OK). 你可以使用 getStatusCode() 方法獲取 PSR-7 響應物件的返回代碼.

```
$status = $response->getStatusCode();
```

當你, 你可以用 PSR-7 響應物件返回指定的狀態碼.

```
$newResponse = $response->withStatus(302);
```

## 響應表頭

每個 HTTP 響應都有表頭. 這些都是敘述 HTTP 響應但主體不可見的內容. PSR-7 響應物件提供了下列的方式查看與操作這些表頭.

- getHeaders(): 取得所有的表頭

你可以使用 PSR-7 響應物件的 getHeaders() 方法取得 HTTP 響應表頭作為關聯陣列. 

```
$headers = $response->getHeaders();
foreach ($headers as $name => $values) {
  echo $name . ": " . implode(", ", $values);
}
```

- getHeader(): 取得單一表頭, 以陣列型態返回

你可以使用 PSR-7 響應物件的 getHeader() 方法取得單組表頭值. 它將會返回一個陣列. 注意一個 HTTP 表頭也可能包含超過一組數值.

```
$header = $newResponse->getHeader('Content-type');
foreach ($header as $key => $value) {
  echo $key . ": " . $value;
}
```

- getHeaderLine(): 取得單一表頭, 以字串型態返回

你還可以使用 PSR-7 響應物件的 getHeaderLine() 方法取得單組表頭值, 它會以逗號方式分隔超過組以上的數值.

```
$header = $newResponse->getHeader('Content-type');
echo $header;
```

- hasHeader(): 查看是否有此表頭

你可以使用 PSR-7 響應物件的 hasHeader() 方法測試某表頭是否存在.

```
if ($newResponse->hasHeader('Access-Control-Allow-Headers')) {
  // ...
}
```

- withHeader(): 設置表頭

你可以使用 PSR-7 響應物件的 withHeader() 方法設置某表頭值. 

```
$newResponse = $response->withHeader('Content-type', 'application/json');
```

> \* 這個響應物件不可變; 這個方法會返回具有新設置的表頭值, 並且具備破壞性, 它將覆蓋相同名稱的表頭值.

- withAddedHeader(): 為某表頭添加

你可以使用 PSR-7 響應物件的 withAddedHeader() 方法添加某表頭額外數值.

```
$newResponse = $response->withAddedHeader('allow', 'post')
  ->withAddedHeader('allow', 'put')
  ->withAddedHeader('allow', 'patch');
```

> \* 與 withHeader() 不同的是, 這個方法不會覆蓋相同名稱的表頭值, 而是附加在既有的表頭值上; 這個響應物件不可變, 並且會返回被變更的響應物件副本.

- withoutHeader(): 移除表頭

你可以使用響應物件的 withoutHeader() 方法移除某表頭值.

```
$newResponse = $response->withoutHeader('allow');
```

> \* 這個響應物件不可變, 並且會返回被變更的響應物件副本.

## 響應主體

HTTP 響應通常都會有主體(Body).

就像 PSR-7 請求物件一樣, PSR-7 響應物件將 Psr\Http\Message\StreamInterface 實作為一個主體的實例. 你可以使用 PSR-7 物件的 getBody() 方法取得響應主體. 對於傳入的 HTTP 響應大小未知或需要大量記憶體, getBody() 方法是更合適的.

```
$body = $response->getBody();
```

Psr\Http\Message\StreamInterface 提供以下的方法來讀取、迭代或是寫入底層的 PHP 資源.

  - getSize(): 取得響應的內容大小
  - tell()
  - eof()
  - isSeekable()
  - seek()
  - rewind()
  - isWritable()
  - write($string): 輸出字串
  - isReadable()
  - read($length)
  - getContents()
  - getMetadata($key = null): 返回與 stream_get_meta_data() 相同資料.

通常, 你需要寫入 PSR-7 響應物件. 你可以使用 write() 方法將內容寫入 StreamInterface 實例, 範例如下：

```
$body = $response->getBody();
$body->write('Hello');
```

你可以透過 StreamInterface 實例去替代 PSR-7 響應物件主體. 當你希望從遠端的內容(如檔案系統或遠端API)透過管道傳送至 HTTP 響應時, 這特別有用. 你可以以 withBody() 方法取代 PSR-7 響應物件主體. 他的參數必須是 Psr\Http\Message\StreamInterface 實例.

```
use GuzzleHttp\Psr7\LazyOpenStream;

$newStream = new LazyOpenStream('/path/to/file', 'r');
$newResponse = $response->withBody($newStream);
```

> \* 這個響應物件不可變, 並且會返回被變更的響應物件副本.

## 返回 JSON

在單純的表單中, JSON 資料可以返回預設 200 的 HTTP 狀態碼.

```
$app->get('/', function (Request $request, Response $response, array $args) {
  $payload = json_encode([
    'name' => 'Bob',
    'age'  => 40,
  ]);

  $response->getBody()->write($payload);
  return $response->withHeader('Content-type', 'application/json');
});
```

你也可已變更回傳的 JSON 使用不同的 HTTP 狀態碼.

```
$app->get('/', function (Request $request, Response $response, array $args) {
  $payload = json_encode([
    'name' => 'Bob',
    'age'  => 40,
  ]);

  $response->getBody()->write($payload);
  return $response->withHeader('Content-type', 'application/json')
    ->withStatus(201);
});
```

> \* 這個響應物件不可變; 這個方法會返回具有新設置的表頭值, 並且具備破壞性, 它將覆蓋相同名稱的表頭值.

## 返回轉址

你可以使用 Location 標頭導向新的 HTTP 網址.

```
$app->get('/', function (Request $request, Response $response, array $args) {
  return $response->withHeader('Location', 'http://localhost:8888/hello')
    ->withStatus(302);
});
```