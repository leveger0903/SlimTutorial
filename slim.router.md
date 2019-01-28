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

## PUT(CREATE) 路由

````
$app = new Slim\App();

$app->put('/books/{id}', function (REQUEST $request, RESPONSE $response, $args) {
  $response->write('put: ' . $args['id']);
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