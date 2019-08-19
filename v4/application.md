# Slim 4 應用

## 概觀 (Overview)

Slim\App 是你的 Slim 應用程序入口點, 用於連接你的回調(callback)、控制器(controller)與註冊路由.

### 基本範例

```
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use Slim\Factory\AppFactory;
  
  require __DIR__ . '/../vendor/autoload.php';
  
  $app = AppFactory::create();
  
  // 加入錯誤中間層
  $app->addErrorMiddleware(true, false, false);
  
  $app->get('/application', function (Request $request, Response $response, array $args) {
    $response->getBody()->write('Hello World');
    return $response;
  });
  
  $app->run();

```

### 進階的錯誤與警告處理

警告與錯誤訊息預設是不會被捕捉的, 如果你希望應用在發生上述類型的錯誤時, 可參考下列實作範例.

```
## index.php
<?php

  use Psr\Http\Message\ResponseInterface as Response;
  use Psr\Http\Message\ServerRequestInterface as Request;
  use MyApp\Handlers\ErrorHandler;
  use MyApp\Handlers\ShutdownHandler;
  use Slim\Exception\HttpInternalServerErrorException;
  use Slim\Factory\AppFactory;
  use Slim\Factory\ServerRequestCreatorFactory;

  require __DIR__ . '/../vendor/autoload.php';
  require __DIR__ . '/HttpErrorHandler.php';
  require __DIR__ . '/ShutdownHandler.php';

  $app = AppFactory::create();

  $request = ServerRequestCreatorFactory::create()
      ->createServerRequestFromGlobals();

  $errorHandler = new MyApp\Handlers\HttpErrorHandler(
    $app->getCallableResolver(), 
    $app->getResponseFactory()
  );

  $displayErrorDetails = true;

  $shutdownHandler = new MyApp\Handlers\ShutdownHandler($request, $errorHandler, $displayErrorDetails);
  register_shutdown_function($shutdownHandler);

  $app->addRoutingMiddleware();

  $errorMiddleware = $app->addErrorMiddleware($displayErrorDetails, false, false);
  $errorMiddleware->setDefaultErrorHandler($errorHandler);

  $app->get('/', function (Request $request, Response $response, array $args) {
    $response->getBody()->write('Hello World');
    return $response;
  });

  $app->run();

## HttpErrorHandler.php
<?php
  
  namespace MyApp\Handlers;

  use Psr\Http\Message\ResponseInterface;
  use Slim\Exception\HttpBadRequestException;
  use Slim\Exception\HttpException;
  use Slim\Exception\HttpForbiddenException;
  use Slim\Exception\HttpMethodNotAllowedException;
  use Slim\Exception\HttpNotFoundException;
  use Slim\Exception\HttpNotImplementedException;
  use Slim\Exception\HttpUnauthorizedException;
  use Slim\Handlers\ErrorHandler;
  use Exception;
  use Throwable;

  class HttpErrorHandler extends ErrorHandler
  {
    public const BAD_REQUEST = 'BAD_REQUEST';
    public const INSUFFICIENT_PRIVILEGES = 'INSUFFICIENT_PRIVILEGES';
    public const NOT_ALLOWED = 'NOT_ALLOWED';
    public const NOT_IMPLEMENTED = 'NOT_IMPLEMENTED';
    public const RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND';
    public const SERVER_ERROR = 'SERVER_ERROR';
    public const UNAUTHENTICATED = 'UNAUTHENTICATED';
    
    protected function respond(): ResponseInterface
    {
      $exception = $this->exception;
      $statusCode = 500;
      $type = self::SERVER_ERROR;
      $description = 'An internal error has occurred while processing your request.';

      if ($exception instanceof HttpException) {
        $statusCode = $exception->getCode();
        $description = $exception->getMessage();

        if ($exception instanceof HttpNotFoundException) {
          $type = self::RESOURCE_NOT_FOUND;
        } elseif ($exception instanceof HttpMethodNotAllowedException) {
          $type = self::NOT_ALLOWED;
        } elseif ($exception instanceof HttpUnauthorizedException) {
          $type = self::UNAUTHENTICATED;
        } elseif ($exception instanceof HttpForbiddenException) {
          $type = self::UNAUTHENTICATED;
        } elseif ($exception instanceof HttpBadRequestException) {
          $type = self::BAD_REQUEST;
        } elseif ($exception instanceof HttpNotImplementedException) {
          $type = self::NOT_IMPLEMENTED;
        }
      }

      if (
        !($exception instanceof HttpException)
        && ($exception instanceof Exception || $exception instanceof Throwable)
        && $this->displayErrorDetails
      ) {
        $description = $exception->getMessage();
      }

      $error = [
        'statusCode' => $statusCode,
        'error' => [
          'type' => $type,
          'description' => $description,
        ],
      ];
        
      $payload = json_encode($error, JSON_PRETTY_PRINT);  
      $response = $this->responseFactory->createResponse($statusCode);        
      $response->getBody()->write($payload);
        
      return $response;
    }
  }

## ShutdownHandler.php
<?php

  namespace MyApp\Handlers;

  use Psr\Http\Message\ServerRequestInterface as Request;
  use MyApp\Handlers\HttpErrorHandler;
  use Psr\Http\Message\ServerRequestInterface;
  use Slim\Exception\HttpInternalServerErrorException;
  use Slim\ResponseEmitter;

  class ShutdownHandler
  {
    private $request;
    private $errorHandler;
    private $displayErrorDetails;

    public function __construct(Request $request, HttpErrorHandler $errorHandler, bool $displayErrorDetails) {
      $this->request = $request;
      $this->errorHandler = $errorHandler;
      $this->displayErrorDetails = $displayErrorDetails;
    }

    public function __invoke()
    {
      $error = error_get_last();
      if ($error) {
        $errorFile = $error['file'];
        $errorLine = $error['line'];
        $errorMessage = $error['message'];
        $errorType = $error['type'];
        $message = 'An error while processing your request. Please try again later.';

        if ($this->displayErrorDetails) {
          switch ($errorType) {
            case E_USER_ERROR:
              $message = "FATAL ERROR: {$errorMessage}. ";
              $message .= " on line {$errorLine} in file {$errorFile}.";
              break;

            case E_USER_WARNING:
              $message = "WARNING: {$errorMessage}";
              break;

            case E_USER_NOTICE:
              $message = "NOTICE: {$errorMessage}";
              break;

            default:
              $message = "ERROR: {$errorMessage}";
              $message .= " on line {$errorLine} in file {$errorFile}.";
              break;
          }
        }

        $exception = new HttpInternalServerErrorException($this->request, $message);
        $response = $this->errorHandler->__invoke($this->request, $exception, $this->displayErrorDetails, false, false);
            
        ob_clean();
        $responseEmitter = new ResponseEmitter();
        $responseEmitter->emit($response);
      }
    }
  }
```