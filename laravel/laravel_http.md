<!-- ---
title: Laravel HTTP 请求处理
date: 2020-12-27 16:28:15
category: showcode, laravel
--- -->

# Laravel HTTP 请求处理

Laravel 对 HTTP 请求的处理流程是：

1. 启动服务容器
2. 绑定处理实例到容器
3. 获取请求参数
4. 处理请求
5. 发送回响应参数


## 1. 启动服务容器

创建应用服务容器，并且绑定服务实例。

```php
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);

$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);
```

获取请求处理kernel，获取请求并进行处理。

```php
// 获取处理kernel
$kernel = $app->make(Kernel::class);

// 请求处理
$response = tap($kernel->handle(
    $request = Request::capture()
))->send();

// 请求终止处理
$kernel->terminate($request, $response);
```

## 2. Http 请求处理

`\Illuminate\Foundation\Http\Kernel::handle` 中处理http 请求。

```php
$kernel->handle(
    $request = Request::capture()
)
```

出入请求参数，处理后响应结果。

```php
public function handle($request)
{
    try {
        // 请求处理
        $response = $this->sendRequestThroughRouter($request);
    } catch (Throwable $e) {
        // 上报异常
        $this->reportException($e);
        // 渲染异常响应
        $response = $this->renderException($request, $e);
    }

    // 分发请求处理时间
    $this->app['events']->dispatch(
        new RequestHandled($request, $response)
    );

    return $response;
}
```

请求处理：

```php
protected function sendRequestThroughRouter($request)
{
    // 服务启动处理，启动应用依赖的服务
    $this->bootstrap();

    // 基于管道的请求处理
    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}
```

应用主要依赖服务启动。

```php
// \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class,
// \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,
// \Illuminate\Foundation\Bootstrap\HandleExceptions::class,
// \Illuminate\Foundation\Bootstrap\RegisterFacades::class,
// \Illuminate\Foundation\Bootstrap\RegisterProviders::class,
// \Illuminate\Foundation\Bootstrap\BootProviders::class,
public function bootstrap()
{
    // ...
    $this->app->bootstrapWith($this->bootstrappers());
}

// \Illuminate\Foundation\Application::bootstrapWith
public function bootstrapWith(array $bootstrappers)
{
    foreach ($bootstrappers as $bootstrapper) {
        // 启动服务
        $this->make($bootstrapper)->bootstrap($this);
    }
}
```

### 基于管道的请求处理

1. 首先构造 Illuminate\Routing\Pipeline 实例。 
2. 然后将当前请求实例 $request 赋值到 Pipeline 的 $passable 属性。 
3. 如果应用启用中间件功能的话（默认启用），则将全局中间件数组 $middleware 赋值到 Pipeline 的 $pipes 属性。
4. 接下来调用 dispatchToRouter 方法将请求转发给路由，该方法返回一个闭包函数，这个闭包函数将在处理完全局中间件逻辑后执行。
5. 接下来调用 Pipeline 的 then 方法。

```php
return (new Pipeline($this->app))
            ->send($request)
            ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
            ->then($this->dispatchToRouter());
```

## 4. 请求终止处理

```php
$kernel->terminate($request, $response);

public function terminate($request, $response)
{
    // 中间件终止调用
    $this->terminateMiddleware($request, $response);
    // 终止回调调用
    $this->app->terminate();
}
```

中间件终止方法调用：

```php
protected function terminateMiddleware($request, $response)
{
    $middlewares = array_merge(
        $this->gatherRouteMiddleware($request),
        $this->middleware
    );

    foreach ($middlewares as $middleware) {
        // ...
        // 解析中间件
        [$name] = $this->parseMiddleware($middleware);
        $instance = $this->app->make($name);

        // 调用中间件terminate 方法
        if (method_exists($instance, 'terminate')) {
            $instance->terminate($request, $response);
        }
    }
}
```

注册的终止回调调用：

```php
public function terminate()
{
    foreach ($this->terminatingCallbacks as $terminating) {
        $this->call($terminating);
    }
}
```

## 参考资料

- https://xueyuanjun.com/post/9804