## 门面模式
**门面模式** 又叫 **外观模式**，提供一个统一的接口去访问多个子系统的多个不同的接口，它为子系统中的一组接口提供一个统一的高层接口，使用子系统更容易使用。

本质：就是化零为整；引入一个中介类，把各个分散的功能组合成一个整体，只对外暴露一个统一的接口；

这两年流行微服务，即化整为零，把一个大服务拆分成一个个零部件；而门面模式则是反其道，是化零为整；

## 目的
为了用户使用方便，把过度拆分的分散功能，组合成一个整体，对外提供一个统一的接口

## Laravel 框架中的门面模式

### 如何使用

`Laravel`  框架通过使用门面设计模式，使得可以通过调用类的静态方法的方式去调用类的普通方法，十分的方便和简单，以  `Cache`  类为例：
```php
public function getCacheData($name)
{
    return Cache::get($name)
}
```
上述代码，其实调用的是  `Cache`  类的非静态  `get`  方法

### 源码解析
这是  `Illuminate\Support\Facades\Cache`  facade 类的源代码：

```php
class Cache extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'cache';
    }
}
```

我们是如何使用这个类的呢？

```php
public function test()
{
   Cache::get('test');
}
```
上述代码看起来非常像调用  `Cache`  中的静态方法  `get` ，但是我们很清楚的看到这个类中只有  `getFacadeAccessor`  这一个方法，并没有静态方法  `get` ，这里的方法 `get()`  实际上存在于容器内部的服务中。所有的细节都隐藏在基本  `Facade`  类中。

当我们在  `Cache`  外观上引用任何静态方法时，`Laravel`  会解析  `cache`  服务容器中的 绑定并针对该对象运行请求的方法。

现在让我们仔细研究一下这个细节，每个外观都将扩展基本抽象  `Facade`  类。细节隐藏在这里的三种方法中：

- `__callStatic()` -简单的PHP魔术方法 
- `getFacadeRoot()` -从IoC容器中获取服务
- `resolveFacadeInstance()` -负责解决服务实例

当我们调用静态方法  `get`  的时候，`Cache` 类中没有此方法，会自动调用父类中的魔术方法如下：
```php
    public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();

        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }

        return $instance->$method(...$args);
    }
```

方法 `getFacadeRoot()` 返回外观背后的服务对象的实例：

```php
    public static function getFacadeRoot()
    {
        return static::resolveFacadeInstance(static::getFacadeAccessor());
    }
```

方法 `resolveFacadeInstance` 复杂解决服务实例，在这里，我们检查传递给对象的参数，然后检查是否已经解决了该服务。如果不是，则只需从容器中检索它：

```php
    protected static function resolveFacadeInstance($name)
    {
        if (is_object($name)) {
            return $name;
        }

        if (isset(static::$resolvedInstance[$name])) {
            return static::$resolvedInstance[$name];
        }

        if (static::$app) {
            return static::$resolvedInstance[$name] = static::$app[$name];
        }
    }
```

上述代码中 `static::$app[$name]` 值得注意的是，其中 `$app` 为应用容器，是一个对象，为什么可以以数组的方式访问变量呢？那是因为容器实现了 `ArrayAccess` 类，这个类使得容器对象有了访问数组的能力，对于 `ArrayAccess` 这个类我前面文章有写过，不了解的，可以去看看哦。
调用 `static::$app[$name]` 会直接去调用容器类的 `offsetGet`的方法：
```php
    public function offsetGet($key)
    {
        return $this->make($key);
    }
```
上述代码，其中 `make` 方法继续去解析和返回我们需要的实例，这部分代码不在本文的讨论范围之内，关于容器技术博客其他文章中有详细的描述。

至此，我们就揭开了 `Laravel` 框架中门面模式的面纱了。

