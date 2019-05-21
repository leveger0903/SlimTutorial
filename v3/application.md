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