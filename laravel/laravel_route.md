<!-- ---
title: Laravel 路由实现
date: 2020-12-27 16:30:10
category: showcode, laravel
--- -->

# Laravel 路由实现

路由匹配和处理逻辑入口位于 vendor/laravel/framework/src/Illuminate/Routing/Router.php 的 dispatch 方法。

主体逻辑分为两部分，首先是通过 findRoute 方法进行路由匹配，然后通过 runRoute 执行对应的路由逻辑。

```php
public function dispatch(Request $request)
{
    // ...
    return $this->dispatchToRoute($request);
}

public function dispatchToRoute(Request $request)
{
    return $this->runRoute($request, $this->findRoute($request));
}
```

## 1. 路由匹配

```php
protected function findRoute($request)
{
    $this->current = $route = $this->routes->match($request);
    // ...
    return $route;
}
```

```php
public function match(Request $request)
{
    // 获取当前请求方法（GET、POST等）下的所有路由定义
    $routes = $this->get($request->getMethod());

    // ...
    $route = $this->matchAgainstRoutes($routes, $request);

    return $this->handleMatchedRoute($request, $route);
}

protected function matchAgainstRoutes(array $routes, $request, $includingMethod = true)
{
    // 通过当前请求实例 $request 从返回的路由数组 $routes 中匹配路由
    return $routes->merge($fallbacks)->first(function (Route $route) use ($request, $includingMethod) {
        return $route->matches($request, $includingMethod);
    });
}

// 匹配路由
public function matches(Request $request, $includingMethod = true)
{
    $this->compileRoute();

    // 根据请求路径URI、请求方法（GET、POST等）、Scheme（HTTP、HTTPS等） 和域名判断请求和路由是否匹配
    foreach ($this->getValidators() as $validator) {
        // ...
        if (! $validator->matches($this, $request)) {
            return false;
        }
    }

    return true;
}

// 返回匹配器
public static function getValidators()
{
    // ...
    return static::$validators = [
        new UriValidator, new MethodValidator,
        new SchemeValidator, new HostValidator,
    ];
}
```

## 2. 路由执行

```php
protected function runRoute(Request $request, Route $route)
{
    $request->setRouteResolver(function () use ($route) {
        return $route;
    });

    // ...
    return $this->prepareResponse($request,
        $this->runRouteWithinStack($route, $request)
    );
}

// 执行路由业务逻辑
protected function runRouteWithinStack(Route $route, Request $request)
{
    // ...
    $middleware = $shouldSkipMiddleware ? [] : $this->gatherRouteMiddleware($route);

    return (new Pipeline($this->container))
                    ->send($request)
                    ->through($middleware)
                    ->then(function ($request) use ($route) {
                        return $this->prepareResponse(
                            $request, $route->run()
                        );
                    });
}

public function run()
{
    // ...
    try {
        if ($this->isControllerAction()) {
            return $this->runController();
        }

        return $this->runCallable();
    } catch (HttpResponseException $e) {
        return $e->getResponse();
    }
}
```

判断如果路由是由控制方法定义，则执行对应的控制器方法并返回结果：

```php
protected function runController()
{
    return $this->controllerDispatcher()->dispatch(
        $this, $this->getController(), $this->getControllerMethod()
    );
}
```

```php
public function dispatch(Route $route, $controller, $method)
{
    $parameters = $this->resolveClassMethodDependencies(
        $route->parametersWithoutNulls(), $controller, $method
    );

    if (method_exists($controller, 'callAction')) {
        return $controller->callAction($method, $parameters);
    }

    return $controller->{$method}(...array_values($parameters));
}
```

调用 prepareResponse 方法处理请求，准备返回响应：

```php
public function prepareResponse($request, $response)
{
    return static::toResponse($request, $response);
}
```

## 参考资料

- laravel/framework/src/Illuminate/Routing/Router.php