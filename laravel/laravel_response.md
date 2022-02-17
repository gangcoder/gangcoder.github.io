<!-- ---
title: Laravel Response 类实现
date: 2020-12-27 16:30:04
category: showcode, laravel
--- -->

# Laravel Response 类实现

Response 类用于表示发送给终端用户的应用响应，Response 的完整类名是 Illuminate\Http\Response，继承自 Symfony 的 HTTP 响应基类 Symfony\Component\HttpFoundation\Response。

请求经过匹配路由处理后，会返回相应的响应实例，最终将其发送给终端用户。

## 1. 创建响应实例

响应类由路由器类 Illuminate\Routing\Router 的 toResponse 方法生成：

```php
public static function toResponse($request, $response)
{
    // ...
    if ($response instanceof Model && $response->wasRecentlyCreated) {
        $response = new JsonResponse($response, 201);
    }

    return $response->prepare($request);
}
```

创建json 响应类：

```php
class JsonResponse extends BaseJsonResponse
{
    public function __construct($data = null, $status = 200, $headers = [], $options = 0)
    {
        $this->encodingOptions = $options;

        parent::__construct($data, $status, $headers);
    }
```

处理响应header：

```php
public function prepare(Request $request)
{
    $headers = $this->headers;
    
    // ...
    // 设置header 参数
    if ('1.0' == $this->getProtocolVersion() && false !== strpos($headers->get('Cache-Control'), 'no-cache')) {
        $headers->set('pragma', 'no-cache');
        $headers->set('expires', -1);
    }

    return $this;
}
```

## 2. 发送响应

```php
public function send()
{
    // 返回header
    $this->sendHeaders();
    // 返回内容
    $this->sendContent();

    return $this;
}
```

返回header：

```php
public function sendHeaders()
{
    // 如果已经发送，就不再发送响应头
    if (headers_sent()) {
        return $this;
    }

    // headers
    foreach ($this->headers->allPreserveCaseWithoutCookies() as $name => $values) {
        $replace = 0 === strcasecmp($name, 'Content-Type');
        foreach ($values as $value) {
            header($name.': '.$value, $replace, $this->statusCode);
        }
    }

    // ...
    // status
    header(sprintf('HTTP/%s %s %s', $this->version, $this->statusCode, $this->statusText), true, $this->statusCode);

    return $this;
}
```

返回内容：

```php
public function sendContent()
{
    // 返回内容
    echo $this->content;
    return $this;
}
```

## 参考资料

- symfony/http-foundation/Response.php