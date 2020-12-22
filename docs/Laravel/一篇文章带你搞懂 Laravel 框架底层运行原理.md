## 前言
知其然知其所以然，刚开始接触框架的时候大不部分人肯定一脸懵逼，不知道如何实现的，没有一定的基础知识，直接去看框架的源码，只会被直接劝退，`Laravel` 框架是一款非常优秀的 `PHP` 框架，这篇文章就是带你彻底搞懂框架的运行原理，好让你在面试的过程中有些谈资（吹牛），学习和研究优秀框架的源码也有助于我们自身技术的提升，接下来系好安全带，老司机要开始开车了！！！
## 准备知识
- 熟悉 php 基本知识，如常见的数组方法，闭包函数的使用，魔术方法的使用
- 熟悉 php 的反射机制和依赖注入
- 熟悉 php 命名空间概念和 compose 自动加载
- 熟悉常见的设计模式，包括但是不限于单例模式，工厂模式，门面模式，注册树模式，装饰者模式等
## 运行原理概述
`Laravel` 框架的入口文件 `index.php`

1、引入自动加载 `autoload.php` 文件

2、创建应用实例，并同时完成了

```
基本绑定($this、容器类Container等等)、

基本服务提供者的注册（Event、log、routing）、

核心类别名的注册(比如db、auth、config、router等)
```

3、开始 `Http` 请求的处理

```
make 方法从容器中解析指定的值为实际的类，比如 $app->make(Illuminate\Contracts\Http\Kernel::class); 解析出来 App\Http\Kernel 

handle 方法对 http 请求进行处理

实际上是 handle 中 sendRequestThroughRouter 处理 http 的请求

首先，将 request 绑定到共享实例

然后执行 bootstarp 方法，运行给定的引导类数组 $bootstrappers，这里是重点，包括了加载配置文件、环境变量、服务提供者、门面、异常处理、引导提供者等

之后，进入管道模式，经过中间件的处理过滤后，再进行用户请求的分发

在请求分发时，首先，查找与给定请求匹配的路由，然后执行 runRoute 方法，实际处理请求的时候 runRoute 中的 runRouteWithinStack 

最后，经过 runRouteWithinStack 中的 run 方法，将请求分配到实际的控制器中，执行闭包或者方法，并得到响应结果
```

4、 将处理结果返回

## 详细源码分析

1、注册自动加载类，实现文件的自动加载

```php
require __DIR__.'/../vendor/autoload.php';
```

2、创建应用容器实例 `Application` （该实例继承自容器类 `Container`）,并绑定核心（web、命令行、异常），方便在需要的时候解析它们

```php
$app = require_once __DIR__.'/../bootstrap/app.php';
```

`app.php` 文件如下：

```php
<?php
// 创建Laravel实例 【3】
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);
// 绑定Web端kernel
$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);
// 绑定命令行kernel
$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);
// 绑定异常处理
$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
// 返回应用实例
return $app;
```

3、在创建应用实例（`Application.php`）的构造函数中，将基本绑定注册到容器中，并注册了所有的基本服务提供者，以及在容器中注册核心类别名

3.1、将基本绑定注册到容器中

```php
    /**
     * Register the basic bindings into the container.
     *
     * @return void
     */
    protected function registerBaseBindings()
    {
        static::setInstance($this);
        $this->instance('app', $this);
        $this->instance(Container::class, $this);
        $this->singleton(Mix::class);
        $this->instance(PackageManifest::class, new PackageManifest(
            new Filesystem, $this->basePath(), $this->getCachedPackagesPath()
        ));
        # 注：instance方法为将...注册为共享实例，singleton方法为将...注册为共享绑定
    }
```

3.2、注册所有基本服务提供者（事件，日志，路由）

```php
    protected function registerBaseServiceProviders()
    {
        $this->register(new EventServiceProvider($this));
        $this->register(new LogServiceProvider($this));
        $this->register(new RoutingServiceProvider($this));
    }
```

3.3、在容器中注册核心类别名
![](https://gitee.com/yefangyong/blog-image/raw/master/png/20201222145306.png)

4、上面完成了类的自动加载、服务提供者注册、核心类的绑定、以及基本注册的绑定

5、开始解析 `http` 的请求

```php
index.php
//5.1
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
//5.2
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
```

5.1、make方法是从容器解析给定值

```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

中的Illuminate\Contracts\Http\Kernel::class 是在index.php 中的$app = require_once __DIR__.'/../bootstrap/app.php';这里面进行绑定的，实际指向的就是App\Http\Kernel::class这个类
```

5.2、这里对 http 请求进行处理

```php
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
```
进入 `$kernel` 所代表的类 `App\Http\Kernel.php` 中，我们可以看到其实里面只是定义了一些中间件相关的内容，并没有 handle 方法

我们再到它的父类 `use Illuminate\Foundation\Http\Kernel as HttpKernel`; 中找 handle 方法，可以看到 handle 方法是这样的

```php
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();
        // 最核心的处理http请求的地方【6】
        $response = $this->sendRequestThroughRouter($request);
    } catch (Exception $e) {
        $this->reportException($e);
        $response = $this->renderException($request, $e);
    } catch (Throwable $e) {
        $this->reportException($e = new FatalThrowableError($e));
        $response = $this->renderException($request, $e);
    }
    $this->app['events']->dispatch(
        new Events\RequestHandled($request, $response)
    );
    return $response;
}
```

6、处理 `Http` 请求（将 `request` 绑定到共享实例，并使用管道模式处理用户请求）

```php
vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php的handle方法
// 最核心的处理http请求的地方
$response = $this->sendRequestThroughRouter($request);

protected function sendRequestThroughRouter($request)
{
    // 将请求$request绑定到共享实例
    $this->app->instance('request', $request);
    // 将请求request从已解析的门面实例中清除（因为已经绑定到共享实例中了，没必要再浪费资源了）
    Facade::clearResolvedInstance('request');
    // 引导应用程序进行HTTP请求
    $this->bootstrap();【7、8】
    // 进入管道模式，经过中间件，然后处理用户的请求【9、10】
    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}
```

7、在 `bootstrap` 方法中，运行给定的 引导类数组 `$bootstrappers`，加载配置文件、环境变量、服务提供者、门面、异常处理、引导提供者，非常重要的一步，位置在 `vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php`

```php
/**
 * Bootstrap the application for HTTP requests.
 *
 * @return void
 */
public function bootstrap()
{
    if (! $this->app->hasBeenBootstrapped()) {
        $this->app->bootstrapWith($this->bootstrappers());
    }
}
```

```php
/**
 * 运行给定的引导类数组
 *
 * @param  string[]  $bootstrappers
 * @return void
 */
public function bootstrapWith(array $bootstrappers)
{
    $this->hasBeenBootstrapped = true;
    foreach ($bootstrappers as $bootstrapper) {
        $this['events']->dispatch('bootstrapping: '.$bootstrapper, [$this]);
        $this->make($bootstrapper)->bootstrap($this);
        $this['events']->dispatch('bootstrapped: '.$bootstrapper, [$this]);
    }
}

/**
 * Get the bootstrap classes for the application.
 *
 * @return array
 */
protected function bootstrappers()
{
    return $this->bootstrappers;
}

/**
 * 应用程序的引导类
 *
 * @var array
 */
protected $bootstrappers = [
    // 加载环境变量
    \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class,
    // 加载config配置文件【重点】
    \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,
    // 加载异常处理
    \Illuminate\Foundation\Bootstrap\HandleExceptions::class,
    // 加载门面注册
    \Illuminate\Foundation\Bootstrap\RegisterFacades::class,
    // 加载在config/app.php中的providers数组里所定义的服务【8 重点】
    \Illuminate\Foundation\Bootstrap\RegisterProviders::class,
    // 记载引导提供者
    \Illuminate\Foundation\Bootstrap\BootProviders::class,
];

```

8、加载 `config/app.php` 中的 `providers` 数组里定义的服务

```php
Illuminate\Auth\AuthServiceProvider::class,
Illuminate\Broadcasting\BroadcastServiceProvider::class,
......

/**
 * 自己添加的服务提供者
 */
\App\Providers\HelperServiceProvider::class,
```

可以看到，关于常用的 `Redis、session、queue、auth、database、Route` 等服务都是在这里进行加载的

9、使用管道模式处理用户请求，先经过中间件进行处理和过滤

```php
return (new Pipeline($this->app))
    ->send($request)
    // 如果没有为程序禁用中间件，则加载中间件（位置在app/Http/Kernel.php的$middleware属性）
    ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
    ->then($this->dispatchToRouter());
}
```

app/Http/Kernel.php

```php
/**
 * 应用程序的全局HTTP中间件
 *
 * These middleware are run during every request to your application.
 *
 * @var array
 */
protected $middleware = [
    \App\Http\Middleware\TrustProxies::class,
    \App\Http\Middleware\CheckForMaintenanceMode::class,
    \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
    \App\Http\Middleware\TrimStrings::class,
    \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
];
```

10、经过中间件处理后，再进行请求分发（包括查找匹配路由）

```php
/**
 * 10.1 通过中间件/路由器发送给定的请求
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Http\Response
 */
 protected function sendRequestThroughRouter($request)
{
    ...
    return (new Pipeline($this->app))
        ...
        // 进行请求分发
        ->then($this->dispatchToRouter());
}
```

```php
/**
 * 10.2 获取路由调度程序回调
 *
 * @return \Closure
 */
protected function dispatchToRouter()
{
    return function ($request) {
        $this->app->instance('request', $request);
        // 将请求发送到应用程序
        return $this->router->dispatch($request);
    };
}
```

```php
/**
 * 10.3 将请求发送到应用程序
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Http\Response|\Illuminate\Http\JsonResponse
 */
 public function dispatch(Request $request)
{
    $this->currentRequest = $request;
    return $this->dispatchToRoute($request);
}
```

```php
 /**
 * 10.4 将请求分派到路由并返回响应【重点在runRoute方法】
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Http\Response|\Illuminate\Http\JsonResponse
 */
public function dispatchToRoute(Request $request)
{   
    // 
    return $this->runRoute($request, $this->findRoute($request));
}
```

```php
/**
 * 10.5 查找与给定请求匹配的路由
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Routing\Route
 */
protected function findRoute($request)
{
    $this->current = $route = $this->routes->match($request);
    $this->container->instance(Route::class, $route);
    return $route;
}
```

```php
/**
 * 10.6 查找与给定请求匹配的第一条路由
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Routing\Route
 *
 * @throws \Symfony\Component\HttpKernel\Exception\NotFoundHttpException
 */
public function match(Request $request)
{
    // 获取用户的请求类型（get、post、delete、put），然后根据请求类型选择对应的路由
    $routes = $this->get($request->getMethod());
    // 匹配路由
    $route = $this->matchAgainstRoutes($routes, $request);
    if (! is_null($route)) {
        return $route->bind($request);
    }
    $others = $this->checkForAlternateVerbs($request);
    if (count($others) > 0) {
        return $this->getRouteForMethods($request, $others);
    }
    throw new NotFoundHttpException;
}
```

到现在，已经找到与请求相匹配的路由了，之后将运行了，也就是 10.4 中的 runRoute 方法

```php
/**
 * 10.7 返回给定路线的响应
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  \Illuminate\Routing\Route  $route
 * @return \Illuminate\Http\Response|\Illuminate\Http\JsonResponse
 */
protected function runRoute(Request $request, Route $route)
{
    $request->setRouteResolver(function () use ($route) {
        return $route;
    });
    $this->events->dispatch(new Events\RouteMatched($route, $request));
    return $this->prepareResponse($request,
        $this->runRouteWithinStack($route, $request)
    );
}
```

```php

/**
 * Run the given route within a Stack "onion" instance.
 * 10.8 在栈中运行路由,先检查有没有控制器中间件，如果有先运行控制器中间件
 *
 * @param  \Illuminate\Routing\Route  $route
 * @param  \Illuminate\Http\Request  $request
 * @return mixed
 */
protected function runRouteWithinStack(Route $route, Request $request)
{
    $shouldSkipMiddleware = $this->container->bound('middleware.disable') &&
                            $this->container->make('middleware.disable') === true;
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
```

```php
 /**
     * Run the route action and return the response.
     * 10.9 最后一步，运行控制器的方法，处理数据
     * @return mixed
     */
    public function run()
    {
        $this->container = $this->container ?: new Container;

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

11、运行路由并返回响应（重点）
可以看到，10.7 中有一个方法是 `prepareResponse`，该方法是从给定值创建响应实例，而 `runRouteWithinStack` 方法则是在栈中运行路由，也就是说，`http` 的请求和响应都将在这里完成。

## 总结
到此为止，整个 `Laravel` 框架的运行流程就分析完毕了，揭开了 `Laravel` 框架的神秘面纱，其中为了文章的可读性，只给出了核心代码，需要大家结合文章自行去阅读源码，需要注意的是必须了解文章中提到的准备知识，这是阅读框架源码的前提和基础，希望大家有所收获，下车！！！

## 参考资料
- https://www.haveyb.com/article/298
- https://learnku.com/docs/laravel-kernel **重点，最好读完这些文章，再来看这篇文章**
- https://learnku.com/docs/laravel-core-concept/5.5