# Slim 3 應用

## 入門應用

````
$app = new \Slim\App();

$app->get('/', function ($request, $response, $args) {
  return $response->withStatus(200)->write('Hello World!'); 
});

$app->run();
````

## 基礎設定

````
$config = [
  'displayErrorDetails'    => true,
  'addContentLengthHeader' => false,
  'db' => [
    'host'   => 'localhost',
    'user'   => '{username}',
    'pass'   => '{password}',
    'dbname' => '{dbname}'
  ],
  'logger' => [
    'name'  => 'slim-app',
    'level' => Monolog\Logger::DEBUG,
    'path'  => __DIR__ . '/../logs/app.log',
  ],
];

$app = new Slim\App([ 'settings' => $config ]);
$container = $app->getContainer();

$app->get('/', function (REQUEST $request, RESPONSE $response, $args) {
  $settings = $this->get('settings')['logger'];
  return $response->withStatus(200)->write('Hello World!');
});

$app->run();
````

## 可用設置

- httpVersion
  - HTTP 協定版本
  - 預設: 1.1

- responseChunkSize
  - 發送到瀏覽器的響應每區塊大小
  - 預設: 4096

- outputBuffering
  - false 不輸出緩衝, 如果是 append 或是 prepend, 則捕獲 echo 或是 print 方法, 並透過路由返回響應.
  - 預設: append

- determineRouteBeforeAppMiddleware
  - 如果為 true, 代表可以你可以在中間層檢查路由變數
  - 預設: false

- displayErrorDetails
  - 如果為 true, 代表可顯示詳細的錯誤資訊
  - 預設: false

- addContentLengthHeader
  - 如果為 true, 將返回 Content-Length 標頭.
  - 預設: true

- routerCacheFile
  - 用於緩存 FastRoute 路由文件名. 必須設置於可寫目錄的有效文件名. 該文件如不存在, 首次運行將使建立該緩存文件.
  - 預設: false