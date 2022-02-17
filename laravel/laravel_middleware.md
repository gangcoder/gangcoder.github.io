<!-- ---
title: Laravel 中间件实现
date: 2020-12-27 16:30:07
category: showcode, laravel
--- -->

# Laravel 中间件实现

每一个进入 Laravel 应用的请求都会被转化为 Illuminate Request 对象，然后经过所有定义的中间件的处理，以及匹配路由对应处理逻辑处理之后，生成一个 Illuminate Response 响应对象，再经过终端中间件的处理，最终返回给终端用户。

## 1. 中间件调用

```php
return (new Pipeline($this->app))
            ->send($request)
            ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
            ->then($this->dispatchToRouter());

public function then(Closure $destination)
{
    // 创建中间件链
    $pipeline = array_reduce(
        array_reverse($this->pipes()), $this->carry(), $this->prepareDestination($destination)
    );

    return $pipeline($this->passable);
}
```

这里先将中间件数组倒序处理 array_reverse($this->pipes) ，在逐一调用 $this->carry()：

```php
protected function carry()
{
    return function ($stack, $pipe) {
        return function ($passable) use ($stack, $pipe) {
            try {
                if (is_callable($pipe)) {
                    return $pipe($passable, $stack);
                }
                // ...
            } catch (Throwable $e) {
                return $this->handleException($passable, $e);
            }
        };
    };
}
```

历完所有全局中间件数组后，carry 函数返回的还是闭包函数，这段处理的意义是将所有全局中间件和路由处理通过统一的闭包函数进行迭代调用。

## 2. 执行中间件

正式执行 $pipeline 所指向的闭包函数：

```php
$pipeline($this->passable);
```

执行完最后一个全局中间件，控制流程进入到执行 $this->prepareDestination($destination) 返回的闭包函数，其中包含了应用到路由的中间件处理逻辑：

```php
$this->dispatchToRouter()

protected function dispatchToRouter()
{
    return function ($request) {
        $this->app->instance('request', $request);

        return $this->router->dispatch($request);
    };
}
```

## 参考资料

- laravel/framework/src/Illuminate/Pipeline/Pipeline.php