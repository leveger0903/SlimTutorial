# Slim 3 概念

## PSR-7

````
require 'vendor/autoload.php';

$app = new Slim\App();
$app->get('/{name}/{weekday}', function ($request, $response, $args) {
  $request = $request->withAttribute('time', '2016-12-25');
  $time    = $request->getAttribute('time');

  $data = [ 
    'name' => $args['name'], 
    'weekday' => $args['weekday'] 
  ];

  $response->write(json_encode($data));
  
  return $response->withHeader(
    'Content-Type',
    'application/json'
  );
});

$app->run();
````

## 中間層

- 把 Middleware 寫在本體

````
# 產生 'BEFORE [ {name} ] AFTER'
$app->add(function ($request, $response, $next) {
  $response->getBody()->write('BEFORE [');
  $response = $next($request, $response);
  $response->getBody()->write('] AFTER');
  return $response;
});

$app->get('/{name}', function ($request, $response, $args) {
  $response->write("Hello, " . $args['name']);
  return $response;
});

$app->run();  
````

- 把 Middleware 寫在外部類別

````
# 外部Class
class ExampleMiddleware
{
  public function __invoke($request, $response, $next)
  {
    $response->getBody()->write('BEFORE [');
    $response = $next($request, $response);
    $response->getBody()->write('] AFTER');
    return $response;
  }
}

# 路由
$app->get('/{name}', function ($request, $response, $args) {
  $response->write("Hello, " . $args['name']);
  return $response;
})->add(new ExampleMiddleware());

$app->run();
````

## 依賴容器 (Dependency Container)

````
$app = new Slim\App();
$container = $app->getContainer();
$container['view'] = new Slim\Views\PhpRenderer('src/template/');

# 使用方法
$app->get('/tickets', function ($request, $response, $args) {

  # 測試服務是否存在
  if($this->has('view'))
    echo 'view service exist';

  # 顯式取得服務
  $response = $this->get('view')->render($response, 'tickets.php');

  # 隱式取得服務
  $response = $this->view->render($response, 'template.php');
  return $response;

});
````