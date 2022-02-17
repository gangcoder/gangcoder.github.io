<!-- ---
title: Laravel 类和文件自动加载
date: 2020-12-27 16:30:17
category: showcode, laravel
--- -->

# Laravel 类和文件自动加载

Composer 负责管理类和文件的加载操作。

在 public/index.php 中可以看到应用在一开始就引入了类加载器：

```php
require __DIR__.'/../vendor/autoload.php';
```

对应的 vendor/autoload.php 文件由 Composer 在初始化的时候生成，代码很简单：

```php
require_once __DIR__ . '/composer/autoload_real.php';
return ComposerAutoloaderInitXXX::getLoader();
```

## getLoader

```php
public static function getLoader()
{
    // 私有的静态属性 $loader，如果类加载器已经初始化过，则直接返回 $loader，以提高应用性能
    if (null !== self::$loader) {
        return self::$loader;
    }

    spl_autoload_register(array('ComposerAutoloaderInitXXX', 'loadClassLoader'), true, true);
    // 静态属性 $loader 进行赋值，获取类加载器
    self::$loader = $loader = new \Composer\Autoload\ClassLoader();
    spl_autoload_unregister(array('ComposerAutoloaderInitXXX', 'loadClassLoader'));

    // 静态初始化，直接引入路由静态文件
    $useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION') && (!function_exists('zend_loader_file_encoded') || !zend_loader_file_encoded());
    if ($useStaticLoader) {
        require_once __DIR__ . '/autoload_static.php';

        // 设置 $loader 实例的 prefixLengthsPsr4、prefixDirsPsr4、prefixesPsr0、classMap 属性
        call_user_func(\Composer\Autoload\ComposerStaticInitXXX::getInitializer($loader));
    }

    // 完成 Composer 包管理器类自动加载注册
    $loader->register(true);

    // 静态文件，不归属于任何类的辅助函数
    if ($useStaticLoader) {
        $includeFiles = Composer\Autoload\ComposerStaticInitXXX::$files;
    }

    // ...
    foreach ($includeFiles as $fileIdentifier => $file) {
        composerRequireXXX($fileIdentifier, $file);
    }

    return $loader;
}
```

类自动加载注册:

```php
public function register($prepend = false)
{
    spl_autoload_register(array($this, 'loadClass'), true, $prepend);
}
```

## loadClass

当 Laravel 应用执行过程中遇到未定义类时，会调用该方法。

```php
public function findFile($class)
{
    // class map lookup
    if (isset($this->classMap[$class])) {
        return $this->classMap[$class];
    }
    
    // ...
    // 查找文件
    $file = $this->findFileWithExtension($class, '.php');

    if (false === $file) {
        // Remember that this class does not exist.
        $this->missingClasses[$class] = true;
    }

    return $file;
}
```

## 四种自动加载方式

四种自动加载方式对应在 composer.json 中，即 autoload 配置项中的 classmap、psr-4、psr-0 和 files 四个配置。

前三个都是维护类的自动加载，最后一个维护的是文件的自动加载。

1. classMap 管理的是完整类名（可能包含命名空间）与文件路径的映射关系
2. prefixDirsPsr4 管理的是命名空间与文件目录的映射关系（遵循 psr-4 规范）
3. prefixLengthsPsr4 管理的是命名空间前缀长度
4. prefixesPsr0 管理也是命名空间与文件目录映射关系
5. files，需要配置完整的文件路径。


## 参考资料

- https://xueyuanjun.com/post/19890