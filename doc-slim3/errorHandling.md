# Slim 3 錯誤處理

## 預設錯誤處理

````
$configuration = [
  'settings' => [
    'displayErrorDetails' => true,
  ],
];

$c   = new Slim\Container($configuration);
$app = new Slim\App($c);
````

## 自訂500錯誤提示

````
# APP 初始化前
$configuration = [
  'settings' => [
    'displayErrorDetails' => true,
  ],
];

$c = new Slim\Container($configuration);  
$c['phpErrorHandler'] = function ($c) {
  return function (REQUEST $request, RESPONSE $response, $exception) use ($c) {
    return $c['response']
      ->withStatus(500)
      ->withHeader('Content-Type', 'text/html')
      ->write('Something went wrong!');
  };
};
$app = new Slim\App($c);  

# APP 初始化後
$app = new Slim\App();  
$c   = $app->getContainer();
$c['phpErrorHandler'] = function ($c) {
  return function ($request, $response, $exception) use ($c) {
    return $c['response']
      ->withStatus(500)
      ->withHeader('Content-Type', 'text/html')
      ->write('Something went wrong!');
  };
}; 
````

## 自訂500錯誤提示(使用外部class)

````
# class檔案
class CustomHandler 
{
  public function __invoke($request, $response, $args) 
  {
    return $response
      ->withStatus(500)
      ->withHeader('Content-Type', 'text/html')
      ->write('Something went wrong!');
  }
}

# index檔案
require 'vendor/autoload.php';

$configuration = [
  'settings' => [
    'displayErrorDetails' => true,
  ],
];

spl_autoload_register(function ($classname) {
  require("classes/" . $classname . ".php");
});

$app = new Slim\App();
$c   = $app->getContainer();
$c['phpErrorHandler'] = function ($c) {
  return new CustomHandler();
};
````

## 關閉Slim錯誤提示, 使用php原生錯誤提示

````
unset($app->getContainer()['phpErrorHandler']);
````

## 自訂 404 錯誤提示

````
$app = new Slim\App();  
$c   = $app->getContainer();
$c['notFoundHandler'] = function ($c) {
  return function ($request, $response) use ($c) {
    return $c['response']
      ->withStatus(404)
      ->withHeader('Content-Type', 'text/html')
      ->write('Page not found');
  };
};
````

## 自訂405錯誤提示(方法不允許)

````
$app = new Slim\App();  
$c   = $app->getContainer();
$c['notAllowedHandler'] = function ($c) {
  return function ($request, $response, $methods) use ($c) {
    return $c['response']
      ->withStatus(405)
      ->withHeader('Allow', implode(', ', $methods))
      ->withHeader('Content-type', 'text/html')
      ->write(sprintf(
        'Method must be one of: %s',
        implode(', ', $methods)
      ));
  };
};
````