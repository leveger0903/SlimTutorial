# Slim 3 烹飪書

## 處理以 / 结尾的路由模式

````
$app = new Slim\App();

# localhost/fm/Nick/ → localhost/fm/Nick
$app->add(function (Request $request, Response $response, callable $next) {
  $uri  = $request->getUri();
  $path = $uri->getPath();

  if($path != '/' && substr($path, -1) == '/') {
    $uri = $uri->withPath(substr($path, 0, -1));
    return $response->withRedirect((string)$uri, 301);
  }

  return $next($request, $response);
});

$app->get('/{name}', function ($request, $response, $args) {  
  $response->write("Hello, " . $args['name']);
  return $response;
});

$app->run();
````

## 檢索IP地址

````
$app = new Slim\App(['settings' => $config]);
$checkProxyHeaders = true;
$trustedProxies = ['10.0.0.1', '10.0.0.2'];
$app->add(new RKA\Middleware\IpAddress($checkProxyHeaders, $trustedProxies));

$app->get('/{name}', function ($request, $response, $args) {
  $ipAddress = $request->getAttribute('ip_address');
  $response->write("IP: " . $ipAddress);
  return $response;
});

$app->run();
````

## 檢索當前路由

````
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

require 'vendor/autoload.php';

# 基礎設定
$config = [
  'displayErrorDetails'    => true,
  'addContentLengthHeader' => false,
  'db' => [
    'host'   => 'localhost',
    'user'   => '{username}',
    'pass'   => '{password}',
    'dbname' => '{dbname}'
  ],
  'determineRouteBeforeAppMiddleware' => true,
];

$app = new Slim\App(['settings' => $config]);
$app->add(function (Request $request, Response $response, callable $next) {
  $route = $request->getAttribute('route');
  $name = $route->getName();
  $groups = $route->getGroups();
  $methods = $route->getMethods();
  $arguments = $route->getArguments();

  # do something with that information
  echo json_encode($arguments);

  return $next($request, $response);
});

$app->get('/{name}', function ($request, $response, $args) {
  $response->write("Hello, " . $args['name']);
  return $response;
});

$app->run();
````

## 使用 Eloquent ORM

index.php

````
require 'vendor/autoload.php';
  
use \Psr\Http\Message\ServerRequestInterface as Request;
use \Psr\Http\Message\ResponseInterface as Response;

# 基礎設定
$config = [
  'displayErrorDetails'    => true,
  'addContentLengthHeader' => false,
  'db' => [
    'driver'    => 'mysql',
    'host'      => 'localhost',
    'dbname'    => '{dbname}'
    'user'      => '{username}',
    'pass'      => '{password}',
    'charset'   => 'utf8',
    'collation' => 'utf8_unicode_ci',
    'prefix'    => '',
  ],
];
  
$app = new Slim\App([
  'settings' => $config
]);

# Eloquent Orm
$container = $app->getContainer();
$container['db'] = function ($container) {
  $capsule = new \Illuminate\Database\Capsule\Manager;
  $capsule->addConnection($container['settings']['db']);
  $capsule->setAsGlobal();
  $capsule->bootEloquent();
  return $capsule;
};

$app->get('/user/{name}', '\Controller\UserApi:read');
$app->post('/user', '\Controller\UserApi:create');
$app->patch('/user', '\Controller\UserApi:update');
$app->delete('/user', '\Controller\UserApi:delete');
$app->run();
````

classes/Controller/UserApi.php

````
namespace Controller;

use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;
use \Interop\Container\ContainerInterface as CONTAINER;

class UserApi
{
  private $db;

  function __construct(CONTAINER $container)
  {
    $this->db = $container->db;
  }

  public function read(REQUEST $request, RESPONSE $response, $args)
  {
    $result = $this->db->table('users')->find(1);
    return $response->withJson($result);
  }

  public function create(REQUEST $request, RESPONSE $response)
  {      
    $result = $this->db->table('users')->insert([
      'name'     => $request->getParsedBody()['name'],
      'birthday' => $request->getParsedBody()['birthday'],
    ]);

    return $response->withJson([
      'status' => $result,
    ]);
  }

  public function update(REQUEST $request, RESPONSE $response)
  {
    $result = $this->db->table('users')
      ->where(
        'id', 
        $request->getParsedBody()['targetId']
      )
      ->update([
        'name'     => $request->getParsedBody()['name'],
        'birthday' => $request->getParsedBody()['birthday'],
      ]);

    return $response->withJson([
      'status' => $result,
    ]);
  }

  public function delete(REQUEST $request, RESPONSE $response)
  {
    $result = $this->db->table('users')
      ->where(
        'id', 
        $request->getParsedBody()['targetId']
      )
      ->delete();

    return $response->withJson([
      'status' => $result,
    ]);
  }
}
````

> 備註

1. 於 composer.json 加入 autoload 設置

````
{
  ...
  "autoload": {
    "psr-4": {
      "": "classes/"
    }
  }
}
````

## CORS (簡易版)

index.php

````
require 'vendor/autoload.php';
  
use \Psr\Http\Message\ServerRequestInterface as Request;
use \Psr\Http\Message\ResponseInterface as Response;

$app = new Slim\App();
$container = $app->getContainer();

# 通用 CORS 設置
$app->options('/{routes:.+}', function (Request $request, Response $response, $args) {
  return $response;
});

$app->add(function (Request $request, Response $response, $next) {
  $response = $next($request, $response);
  return $response
    ->withHeader('Access-Control-Allow-Origin', 'http://localhost')
    ->withHeader('Access-Control-Allow-Headers', 'X-Requested-With, Content-Type, Accept, Origin, Authorization')
    ->withHeader('Access-Control-Allow-Methods', 'GET, POST, DELETE, PATCH');
});

$app->get('/user/{name}', '\Controller\UserApi:read');
$app->post('/user', '\Controller\UserApi:create');
$app->patch('/user', '\Controller\UserApi:update');
$app->delete('/user', '\Controller\UserApi:delete');

# 設置 404
$container['notFoundHandler'] = function ($container) {
  return new \Handler\NotFound($container);
};

$app->map(['GET', 'POST', 'DELETE', 'PATCH'], '/{routes:.+}', function (Request $request, Response $response) {
  $handler = $this->notFoundHandler;
  return $handler($request, $response);
});

$app->run();
````

classes/Controller/UserApi.php

````
namespace Controller;

use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

class UserApi
{
  public function read(REQUEST $request, RESPONSE $response, $args)
  {
    return $response->withJson([
      'method' => 'select',
    ]);
  }

  public function create(REQUEST $request, RESPONSE $response)
  {      
    return $response->withJson([
      'method' => 'insert',
    ]);
  }

  public function update(REQUEST $request, RESPONSE $response)
  {
    return $response->withJson([
      'method' => 'update',
    ]);
  }

  public function delete(REQUEST $request, RESPONSE $response)
  {
    return $response->withJson([
      'method' => 'delete',
    ]);
  }
}
````

classes/Handler/NotFound.php

````
namespace Handler;

use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

class NotFound 
{
  public function __invoke(REQUEST $request, RESPONSE $response) 
  {
    return $response
      ->withJson(['ststus' => 404])
      ->withStatus(404);
  }
}
````

## CORS (簡易版)

## 模擬環境設定
## 使用 POST 方法上傳檔案
## 動作域響應器(Action-Domain-Responder)
