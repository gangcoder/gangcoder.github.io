<!-- ---
title: Laravel Request 类实现
date: 2020-12-27 16:30:00
category: showcode, laravel
--- -->

# Laravel Request 类实现

捕获请求数据。

```
$request = Request::capture()
```

Request 对象的完整类名是 Illuminate\Http\Request，而该请求类又继承自 Symfony 的 Symfony\Component\HttpFoundation\Request。


## 1. 捕获请求

```php
// \Illuminate\Http\Request::capture
public static function capture()
{
    // 获取请求数据
    return static::createFromBase(SymfonyRequest::createFromGlobals());
}
```

```php
public static function createFromGlobals()
{
    $request = self::createRequestFromFactory($_GET, $_POST, [], $_COOKIE, $_FILES, $_SERVER);
    $request->request = new InputBag($_POST);
    
    // ...
    return $request;
}

private static function createRequestFromFactory(array $query = [], array $request = [], array $attributes = [], array $cookies = [], array $files = [], array $server = [], $content = null): self
{
    // ...
    return new static($query, $request, $attributes, $cookies, $files, $server, $content);
}

public function __construct(array $query = [], array $request = [], array $attributes = [], array $cookies = [], array $files = [], array $server = [], $content = null)
{
    $this->initialize($query, $request, $attributes, $cookies, $files, $server, $content);
}

// 获取请求参数，创建request
public function initialize(array $query = [], array $request = [], array $attributes = [], array $cookies = [], array $files = [], array $server = [], $content = null)
{
    $this->request = new ParameterBag($request);
    $this->query = new InputBag($query);
    $this->attributes = new ParameterBag($attributes);
    $this->cookies = new InputBag($cookies);
    $this->files = new FileBag($files);
    $this->server = new ServerBag($server);
    $this->headers = new HeaderBag($this->server->getHeaders());
}
```

laravel request 扩展request 信息：

```php
public static function createFromBase(SymfonyRequest $request)
{
    $newRequest = (new static)->duplicate(
        $request->query->all(), $request->request->all(), $request->attributes->all(),
        $request->cookies->all(), $request->files->all(), $request->server->all()
    );

    $newRequest->headers->replace($request->headers->all());
    $newRequest->content = $request->content;
    $newRequest->request = $newRequest->getInputSource();

    return $newRequest;
}
```

## 2. 获取请求数据

`$request->get('param');` 获取参数：

```php
public function get(string $key, $default = null)
{
    return parent::get($key, $default);
}

public function get(string $key, $default = null)
{
    // ...
    if ($this->query->has($key)) {
        return $this->query->all()[$key];
    }

    if ($this->request->has($key)) {
        return $this->request->all()[$key];
    }

    return $default;
}
```


## 参考资料

- https://xueyuanjun.com/post/9818