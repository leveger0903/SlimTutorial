# Slim 3 基礎

## 基本初始

````
require 'vendor/autoload.php';

$app = new Slim\App();

$app->get('/{name}/{weekday}', function ($request, $response, $args) {
  $response->write("Hello, " . $args['name'] . ", this is " . $args['weekday']);
  return $response;
});

$app->run();
````

## 加入 config 設定

````
require 'vendor/autoload.php';
$config = [
  'displayErrorDetails'    => true,
  'addContentLengthHeader' => false,
  'db' => [
    'host'   => 'localhost',
    'user'   => '{username}',
    'pass'   => '{password}',
    'dbname' => '{dbname}'
  ]
];
$app = new Slim\App([ 'settings' => $config ]);
````

## 自動載入自己的 class

````
spl_autoload_register(function ($classname) {
  require("classes/" . $classname . ".php");
});

$mySampleClass = new myCustomClass();
echo $mySampleClass->response();
````

## 透過 composer.json 自動載入自己的類

```
{
    "require": {
        "slim/slim": "^3.1",
        "slim/php-view": "^2.0",
        "monolog/monolog": "^1.17",
        "robmorgan/phinx": "^0.5.1"
    },
    "autoload": {
        "psr-4": {
            "": "classes/"
        }
    }
}
```

## 加入依賴容器注入 (Dependency Injection Container)

````
$container = $app->getContainer();

$container['logger'] = function ($config) {
  $logger = new \Monolog\Logger('my_logger');
  $file_handler = new \Monolog\Handler\StreamHandler('logs/app.log');
  $logger->pushHandler($file_handler);
  return $logger;
};

$container['db'] = function ($config) {
  $db = $config['settings']['db'];
  $pdo = new PDO("mysql:host=" . $db['host'] . ";dbname=" . $db['dbname'], $db['user'], $db['pass']);
  $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
  $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
  return $pdo;
};
````

## 取得 GET 資料

````
$app->get('/{name}', function ($request, $response, $args) {  
  $response->write("Hello, " . $args['name']);
  return $response;
});

# 可以透過此方法取得所有GET資訊
$data = $request->getQueryParams();
````

## 取得 POST 資料

````
$app->post('/post', function ($request, $response) {

$post = $request->getParsedBody();
$data = [
  'title' => filter_var($post['title'], FILTER_SANITIZE_STRING),
  'money' => filter_var($post['money'], FILTER_SANITIZE_STRING),
];

$response->write("Hello, " . $data['title'] . ", I'll give you $" . $data['money']);
  return $response;
});

# 可以透過此方法取得所有POST資訊
$data = $request->getParsedBody();
````