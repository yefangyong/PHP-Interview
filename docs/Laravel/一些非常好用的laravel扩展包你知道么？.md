##  前言
自从有了 `composer` 包管理工具，出现大量优秀的扩展包，让我们可以解放双手，大大提高我们的开发效率，有了更多的时间去 `enjoy life and accompany family`，下面我将会列出工作中用到的一些扩展包，希望对大家有所帮助，我不会给出详细的使用说明，大家在使用之前最好先去看下官方的文档，以文档为主，`show time，enjoy`！！！

##  扩展包

### 1、`PHP` 导出百万级数据到表格

**简介：**

PHP 导出是一个比较常见的功能，但常规的导出却有一个内存瓶颈，导致速度慢，甚至会将整个服务给挂掉。

这里，采用了PHP 迭代器 yield，开发了一个简单的 composer 包，使用起来比较简单，导出百万级数据，不会拖慢整个服务。

**使用：**

```
composer require haveyb/export-csv
```

**地址：**

> https://packagist.org/packages/haveyb/export-csv

### 2、`jwt`扩展包

**简介：**

`Json web token (JWT)`, 是为了在网络应用环境间传递声明而执行的一种基于JSON的开放标准（(RFC 7519).该 token 被设计为紧凑且安全的，特别适用于分布式站点的单点登录（`SSO`）场景。`JWT` 的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源，也可以增加一些额外的其它业务逻辑所必须的声明信息，该token也可直接被用于认证，也可被加密。


**使用：**

```
composer require tymon/jwt-auth
```

**地址：**

> https://github.com/tymondesigns/jwt-auth

### 3、`laravel` 框架代码提示扩展包 `laravel-ide-helper`

**简介：**

在使用模型或者门面的时候，编辑器无法自动提示模型有哪些属性和方法，这个扩展包直接从源代码完善PHP注释，以致于编辑器可以自动提示模型或者门面有哪些属性和方法。

**使用：**

```
composer require --dev barryvdh/laravel-ide-helper
php artisan ide-helper:generate - PHPDoc generation for Laravel Facades
php artisan ide-helper:models - PHPDocs for models
php artisan ide-helper:meta - PhpStorm Meta file
```

**地址：**

> https://github.com/barryvdh/laravel-ide-helper

### 4、图片生成或者裁剪扩展包 `BaconQrCode`

**简介：**

在日常开发的过程中，我们不可避免的会遇到生成二维码或者图片的需求，这些代码编写起来比较复杂，但是我们不必重复造轮子，可以直接使用 `BaconQrCode` 扩展包

**使用：**

```
composer require bacon/bacon-qr-code
```

**地址：**

> https://github.com/Bacon/BaconQrCode

### 5、微信支付和支付宝支付扩展包

**简介：**

在电商系统中，微信支付和支付宝支付是一个不可逃避的话题，开发了多次支付宝与微信支付后，很自然产生一种反感，惰性又来了，想在网上找相关的轮子，可是一直没有找到一款自己觉得逞心如意的，要么使用起来太难理解，要么文件结构太杂乱，只有自己撸起袖子干了。

**使用：**

```
  composer require yansongda/pay
  ```

**地址：**

> https://github.com/yansongda/pay

**欢迎大家补充，持续更新中**。。。。。。