## jwt 是什么？

`Json web token (JWT)`, 是为了在网络应用环境间传递声明而执行的一种基于JSON的开放标准（(RFC 7519).该 token 被设计为紧凑且安全的，特别适用于分布式站点的单点登录（`SSO`）场景。`JWT` 的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源，也可以增加一些额外的其它业务逻辑所必须的声明信息，该token也可直接被用于认证，也可被加密。

## 为什么不使用 session，而直接使用jwt呢？

- `session` 存储在服务器端，而 `Jwt` 存储在客户端。
- `session` 方式存储用户信息的最大问题在于要占用服务器内存，增加服务器的开销，而 `Jwt` 方式将用户状态分散到了客户端中，可以明显减轻服务器的内存压力。

## 如何使用

1、`composer` 安装

```
composer require tymon/jwt-auth
```

2、注册服务提供者，`laravel` 框架版本小于5.4时，需要手动在 `config/app.php` 文件中注册服务提供者

```
'providers' => [

    ...

    Tymon\JWTAuth\Providers\LaravelServiceProvider::class,
]

``` 

3、生成配置文件，执行以下命令

```
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
```

4、生成 `jwt` 密钥

```
php artisan jwt:secret
```
你也可以在 `.env` 文件中修改密钥，比如  JWT_SECRET=xxxx

5、修改 `auth.php` 文件中的配置，修改 `Laravel` 框架默认的 `auth` 驱动

```
'defaults' => [
    'guard' => 'api',
    'passwords' => 'users',
],

...

'guards' => [
    'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
    ],
],

'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model'  => App\Models\User\User::class
        ],

        // 'users' => [
        //     'driver' => 'database',
        //     'table' => 'users',
        // ],
    ],
```

上述配置，表示 `auth` 驱动使用 `jwt` 的方式，用户提供者使用 `users`,用户提供者支持 `database` 和 `eloquent` 两种方式

6、修改 `User` 模型，实现 `getJWTIdentifier()` 和 `getJWTCustomClaims()`

```php
<?php

namespace App;

use Tymon\JWTAuth\Contracts\JWTSubject;
use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable implements JWTSubject
{
    use Notifiable;

    // Rest omitted for brevity

    /**
     * Get the identifier that will be stored in the subject claim of the JWT.
     *
     * @return mixed
     */
    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    /**
     * Return a key value array, containing any custom claims to be added to the JWT.
     *
     * @return array
     */
    public function getJWTCustomClaims()
    {
        return [];
    }
}
```

7、添加控制器认证中间件 

```php
    /**
     * Create a new AuthController instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware('auth:api', ['except' => ['login']]);
    }
```

上述代码表示，每次请求都需要经过 `auth` 这个中间件，登录方法除外

## 常用方法

官方文档写的很清楚，这里直接给出官方的说明文档，地址如下：

> https://jwt-auth.readthedocs.io/en/develop/auth-guard/

## 源码解析 (重点)

这里会对源码进行分析，默认你了解一些基本的知识和框架的运行流程，比如容器，依赖注入，门面模式等知识，这里不会对这些知识和内容过多的阐述，如果有不懂和不理解的地方，欢迎评论，大家系好安全带，老司机要发车了！！！

### 用户鉴权

当访问控制器中某个方法时，会先经过 `Authenticate` 中间件进行处理

```php
    //1.1 经过 handle 方法处理请求
    public function handle($request, Closure $next, ...$guards)
    {
        $this->authenticate($request, $guards);

        return $next($request);
    }
```

```php
    //1.2 经过 authenticate 方法检查用户是否登录
     protected function authenticate($request, array $guards)
        {
            if (empty($guards)) {
                $guards = [null];
            }
    
            foreach ($guards as $guard) {
                // 访问 JwtGuard 中的 check 方法 
                if ($this->auth->guard($guard)->check()) { // $this->auth->guard($guard) 会生成JwtGuard对象 
                    return $this->auth->shouldUse($guard);
                }
            }
    
            $this->unauthenticated($request, $guards);
        }
```

```php
    //1.3 访问代码块 GuardHelpers 中的 check 方法，校验用户是否登录
    public function check()
    {
        return ! is_null($this->user());
    }
```

```php
    //1.4 访问 JwtGuard 中的 user 方法 
     public function user()
    {
        if ($this->user !== null) {
            return $this->user;
        }
        //获取请求头中的 token，校验 token 是否过期或者在黑名单中，具体的代码请查看源码
        if ($this->jwt->setRequest($this->request)->getToken() &&
            ($payload = $this->jwt->check(true)) &&
            $this->validateSubject()
        ) {
           //根据负载 payload 获取用户的信息，然后返回
            return $this->user = $this->provider->retrieveById($payload['sub']);
        }
    }
```

```php
    //1.5 创建用户模型，根据主键获取用户信息并且返回
    public function retrieveById($identifier)
    {
        $model = $this->createModel();

        return $this->newModelQuery($model)
                    ->where($model->getAuthIdentifierName(), $identifier)
                    ->first();
    }
```

至此，用户鉴权的核心流程就分析完毕了，这里只给出了核心代码，需要大家根据上述代码去查看源码，只有知其然知其所以然，才可以放心大胆的使用扩展包，出现问题也才可以及时的发现和解决，下车！！！







