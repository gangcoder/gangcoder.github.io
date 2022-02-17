<!-- ---
title: Laravel 异常处理
date: 2020-12-27 16:30:13
category: showcode, laravel
--- -->

# Laravel 异常处理

## 1. 异常处理

用户请求处理过程中的所有未处理异常都会被捕获并按照以下两步进行处理：

1. 根据应用配置向不同的渠道报告异常；
2. 将异常转化为可渲染的格式并以响应的方式返回给终端用户。

```php
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } catch (Throwable $e) {
        $this->reportException($e);

        $response = $this->renderException($request, $e);
    }
    // ...

    return $response;
}
```

报告异常:

```php
protected function reportException(Throwable $e)
{
    $this->app[ExceptionHandler::class]->report($e);
}
```

将异常转化为可渲染的格式:

```php
protected function renderException($request, Throwable $e)
{
    return $this->app[ExceptionHandler::class]->render($request, $e);
}
```

## 2. 默认异常处理注册

Laravel 在初始化时，通过 \Illuminate\Foundation\Bootstrap\HandleExceptions 注册默认异常处理handler。

```php
public function bootstrap(Application $app)
{
    // ...
    error_reporting(-1);

    set_error_handler([$this, 'handleError']);
    set_exception_handler([$this, 'handleException']);
}

public function handleError($level, $message, $file = '', $line = 0, $context = [])
{
    if (error_reporting() & $level) {
        throw new ErrorException($message, 0, $level, $file, $line);
    }
}

// 处理异常
public function handleException(Throwable $e)
{
    try {
        // 报告异常
        $this->getExceptionHandler()->report($e);
    } catch (Exception $e) {
        //
    }

    // 生成异常对应的响应
    if ($this->app->runningInConsole()) {
        $this->renderForConsole($e);
    } else {
        $this->renderHttpResponse($e);
    }
}
```

## 3. 异常处理实现

Laravel 默认的异常处理器位于 app/Exceptions/Handler.php，对应的处理器类继承自基类 Illuminate\Foundation\Exceptions\Handler。

报告异常：

```php
public function report(Throwable $e)
{
    // ...

    // 默认实现时记录到日志中
    try {
        $logger = $this->container->make(LoggerInterface::class);
    } catch (Exception $ex) {
        throw $e;
    }

    $logger->error(
        $e->getMessage(),
        array_merge(
            $this->exceptionContext($e),
            $this->context(),
            ['exception' => $e]
        )
    );
}
```

处理异常响应：

```php
protected function renderHttpException(HttpExceptionInterface $e)
{
    // 响应视图
    if (view()->exists($view = $this->getHttpExceptionView($e))) {
        return response()->view($view, [
            'errors' => new ViewErrorBag,
            'exception' => $e,
        ], $e->getStatusCode(), $e->getHeaders());
    }

    // 影响异常信息
    return $this->convertExceptionToResponse($e);
}
```

## 参考资料

- laravel/framework/src/Illuminate/Foundation/Exceptions/Handler.php