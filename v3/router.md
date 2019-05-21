# Slim 3 路由

## GET(READ) 路由

````
$app = new Slim\App();

$app->get('/books/{id}', function (REQUEST $request, RESPONSE $response, $args) {
  $response->write('get: ' . $args['id']);
});

$app->run();
````

## POST(CREATE) 路由

````
$app = new Slim\App();
$app->post('/books', function (REQUEST $request, RESPONSE $response) {

  $post = $request->getParsedBody();
  $data = [
    'name' => filter_var($post['name'], FILTER_SANITIZE_STRING),
  ];
  return $response->write('post: ' . $data['name']);

});

$app->run();
````

## PUT(CREATE+UPDATE) 路由

````
$app = new Slim\App();

$app->put('/books/{id}', function (REQUEST $request, RESPONSE $response, $args) {

  $response->write('put: ' . $args['id']);

});

$app->run();
````

## PATCH 路由(UPDATE)

````
$app = new Slim\App();

$app->any('/books/[{id}]', function (REQUEST $request, RESPONSE $response, $args) {

  $response->write('any: ' . $args['id']);

});

$app->run();
````

## DELETE 路由

````
$app = new Slim\App();

$app->delete('/books/{id}', function (REQUEST $request, RESPONSE $response, $args) {
  $response->write('delete: ' . $args['id']);
});

$app->run();
````

## OPTIONS 路由

````
$app = new Slim\App();

$app->options('/books/{id}', function (REQUEST $request, RESPONSE $response, $args) {

  $response->write('options: ' . $args['id']);

});

$app->run();
````

## 任意路由

````
$app = new Slim\App();

$app->any(
  '/books/[{id}]', 
  function (REQUEST $request, RESPONSE $response, $args) {
    $response->write('any: ' . $args['id']);
  }
);

$app->run();
````

## 多重路由

````
$app = new Slim\App();

$app->map(
  ['GET', 'PUT', 'DELETE'], 
  '/books/{id}', 
  function (REQUEST $request, RESPONSE $response, $args) {
    $response->write('map:' . $args['id']);
  }
);

$app->run();
````

## 路由使用自訂義class代替閉包實作(需使用__invoke())

````
# class檔案
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

class MyRestfulController 
{
  public function __invoke(REQUEST $request, RESPONSE $response, $args)
  {
    return $response->write('get: ' . $args['id']);
  }
}

# index檔案
$app = new Slim\App();

spl_autoload_register(function ($classname) {
  require("classes/" . $classname . ".php");
});

$app->get('/books/{id}', 'MyRestfulController');
$app->run();
````

## 閉包綁定

````
$app = new Slim\App();

# 實作DI(參考) concepts 章節
$container = $app->getContainer();
$container['cookies'] = new Slim\Http\Cookies();

$app->get('/hello/{name}', function (REQUEST $request, RESPONSE $response, $args) { 

    $this->cookies->set('name', [
        'name' => $args['name'],
        'expires' => '7 days'
    ]);

});

$app->run();
````

## 路由參數提出

````
$DI = new Slim\Container();
$DI['foundHandler'] = function() {
  return new Slim\Handlers\Strategies\RequestResponseArgs();
};
  
$app = new Slim\App($DI);
$app->get(
  '/hello/{name}/{id}', 
  function (REQUEST $request, RESPONSE $response, $name, $id) { 
    return $response->write(sprintf('name: %s, id: %s', $name, $id));
  }
);
````

## 自訂義路由

````
# 自定義路由(id可選)
$app->get(
  '/news[/{id}]', 
  function (REQUEST $request, RESPONSE $response, $args) { 
    if(!isset($args['id']))
      $args['id'] = 0;

    return $response->write($args['id']);
  }
);

# 自定義路由(year/month可選)
$app->get(
  '/list[/{year}[/{month}]]', 
  function (REQUEST $request, RESPONSE $response, $args) { 

    if(!isset($args['year']))
      $args['year']  = 2017;

    if(!isset($args['month']))
      $args['month'] = 1;

    return $response->write(sprintf('%s-%s', $args['year'], $args['month']));
  }
);

# 自定義路由(選項無限制)
$app->get(
  '/birthday[/{params:.*}]', 
  function (REQUEST $request, RESPONSE $response, $args) {
    
    $params = explode('/', $request->getAttribute('params'));
    return $response->write(json_encode($params));

  }
);

# 自訂義路由(正規化)
$app->get(
  '/phone-number/{number:[0-9]+}', 
  function (REQUEST $request, RESPONSE $response, $args) {

  return $response->write($args['number']);

});
````

## 取得路由名稱

````
$app->get(
  '/hello/{name}', 
  function (REQUEST $request, RESPONSE $response, $args) {
    return $response->write("Hello, " . $args['name']);
  }
)->setName('hello');

$app->get(
  '/work/{name}', 
  function (REQUEST $request, RESPONSE $response, $args) {
    return $response->write(
      $this->router
        ->pathFor(
          'hello', 
          ['name' => $args['name']]
      )
    );
  }
);
````

## 取得路由

````
$app->group('/users/{id:[0-9]+}', function () {

  # localhost/users/123456
  $this->map(
    ['GET', 'DELETE', 'PATCH', 'PUT'], 
    '', 
    function (REQUEST $request, RESPONSE $response, $args) { 
      return $response->write('basic: ' . $args['id']);
    }
  )->setName('user');

  # localhost/users/123456/reset-password
  $this->get(
    '/reset-password', 
    function (REQUEST $request, RESPONSE $response, $args) { 
      return $response->write('reset-password: ' . $args['id']);
    }
  )->setName('user-password-reset');

  # localhost/users/123456/profile
  $this->get(
    '/profile', 
    function (REQUEST $request, RESPONSE $response, $args) {
      return $response->write('profile: ' . $args['id']);
    }
  )->setName('user-profile');

});
````

## 容器辨別(Container Resolution)：方法1, 使用__invoke

````
# Class：HomeController
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;
use \Interop\Container\ContainerInterface as ContainerInterface;  

class HomeController 
{
  protected $container;
    
  # constructor
  public function __construct(ContainerInterface $container) 
  {
    $this->container = $container;
  }
   
  public function __invoke(REQUEST $request, RESPONSE $response, $args) 
  {
    # to access items in the container... $this->container->get('');
    return $response->write('id:' . $args['id']);
  }
}

# index
require 'vendor/autoload.php';
  
$app = new Slim\App();
spl_autoload_register(function ($classname) {
  require("classes/" . $classname . ".php");
});

$app->get('/home/{id}', '\HomeController');
$app->run();
````

## 容器辨別(Container Resolution)：方法2, 多組路由

````
# Class：HomeController
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;
use \Interop\Container\ContainerInterface as ContainerInterface;  

class HomeController 
{

  protected $container;
    
  # constructor
  public function __construct(ContainerInterface $container) 
  {
    $this->container = $container;
  }
   
  public function home(REQUEST $request, RESPONSE $response, $args) 
  {
    # to access items in the container... $this->ci->get('');
    return $response->write('[route]home: ' . $args['id']);
  }

  public function name(REQUEST $request, RESPONSE $response, $args) 
  {
    # to access items in the container... $this->ci->get('');
    return $response->write('[route]name: ' . $args['name']);
  }

}

# index
require 'vendor/autoload.php';
  
$app = new Slim\App();

spl_autoload_register(function ($classname) {
  require("classes/" . $classname . ".php");
});

$app->get('/home/{id}', '\HomeController:home');
$app->get('/name/{name}', '\HomeController:name');  

$app->run();
````

## 容器辨別(Container Resolution)：方法3, 將Class註冊為container

````
# Class：HomeController
use \Interop\Container\ContainerInterface as ContainerInterface;

class HomeController 
{
  protected $container;
    
  # constructor
  public function __construct(ContainerInterface $container) 
  {
    $this->container = $container;
  }
   
  public function home($request, $response, $args) 
  {
    return $response->write('[home]id: ' . $args['id']);
  }

  public function name($request, $response, $args) 
  {
    return $response->write('[name]name: ' . $args['name']);
  }
}

# index  
use \Psr\Http\Message\ServerRequestInterface as REQUEST;
use \Psr\Http\Message\ResponseInterface as RESPONSE;

require 'vendor/autoload.php';
  
$app = new Slim\App();

spl_autoload_register(function ($classname) {
  require("classes/" . $classname . ".php");
});

$container = $app->getContainer();
$container['HomeController'] = function ($c) {
  return new HomeController($c);
};

$app->get(
  '/home/{id}', 
  function (REQUEST $request, RESPONSE $response, $args) {
    return $this->HomeController->home($request, $response, $args);
  }
);

$app->get(
  '/name/{name}', 
  function (REQUEST $request, RESPONSE $response, $args) {
    return $this->HomeController->name($request, $response, $args);
  }
);

$app->run();
````