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