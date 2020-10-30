### CGI协议与FastCGI协议

每种动态语言（`PHP,Python`等）的代码文件需要通过对应的解析器才能被服务器识别，而 `CGI（Common Gateway Interface）` 通用网关接口协议就是用来使解释器和服务器可以互相通信。PHP文件在服务器上的解析需要用到PHP解析器，再加上对应的 `CGI` 协议，从而可以使服务器可以解析PHP文件。

由于CGI的机制是每次处理一个请求，fork一个进程，等请求处理完成后，再去 `kill` 这个进程，在实际应用中比较浪费资源，于是就出现了 `CGI` 的改良版本 `FastCGI` ，`FastCGI` 在请求处理完成后不会马上 `kill` 进程，而是会接着执行其他的请求，这样会大大的提高效率。

### PHP-FPM是什么？

`PHP-FPM` 即 `PHP-FastCGI Process Manager`，它是 `FastCGI` 的一个具体实现，并且提供了进程管理的功能，进程包括 `master` 和 `worker` 两种进程。`master` 进程只有一个，负责监听端口，接受来自服务器的请求，而 `worker` 进程一般有多个，每个进程内部都是嵌入PHP解释器，是代码真正执行的地方。

### Nginx 与 php-fpm 通信机制

当我们访问一个网站（www.test.com）的时候，处理的流程一般是这样的：

![](https://gitee.com/yefangyong/blog-image/raw/master/png/20200908115619.png)

### Nginx 与 php-fpm 的结合

在 `Linux` 上，`nginx` 与 `php-fpm` 通信有两种方式，`tcp-socket` 和 `unix-socket`。

`tcp-socket` 的优点是可以跨服务器，而当 `nginx` 和 `php-fpm` 不在同一台机器上时，只能使用这种方式。

`unix-socket` 又叫IPC(`inter-process communication`) 进程间通信 `socket`，用于实现同一主机的进程间通信，这种方式需要在nginx配置文件中填写 `php-fpm` 的 `socket` 文件位置。

两种方式的数据传输过程如下图：

![](https://gitee.com/yefangyong/blog-image/raw/master/png/20200908120534.png)

二者的不同：

由于 `Unix socket` 不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。所以其效率比 `tcp socket` 的方式要高，可减少不必要的 tcp 开销。不过，`unix socket` 高并发时不稳定，连接数爆发时，会产生大量的长时缓存，在没有面向连接协议的支撑下，大数据包可能会直接出错不返回异常。而 `tcp` 这样的面向连接的协议，可以更好的保证通信的正确性和完整性。

`Nginx` 与 `php-fpm` 结合只需要在各自的配置文件中做设置即可：

1）Nginx中的配置，以 `tcp socket` 通信为例

```shell script
server {
    listen       80; #监听 80 端口，接收http请求
    server_name  www.test.com; #就是网站地址
    root /usr/local/etc/nginx/www/project; # 准备存放代码工程的路径
    #路由到网站根目录 www.test.com 时候的处理
    location / {
        index index.php; #跳转到 www.test.com/index.php
        autoindex on; #开启文件索引目录
    }   

    #当请求网站下 php 文件的时候，反向代理到 php-fpm
    location ~ \.php$ {
        include /usr/local/etc/nginx/fastcgi.conf; #加载 nginx 的 fastcgi 模块
        fastcgi_intercept_errors on;
        fastcgi_pass   127.0.0.1:9000; # tcp 方式，php-fpm 监听的 IP 地址和端口
       # fasrcgi_pass /usr/run/php-fpm.sock # unix socket 连接方式
    }

}
```

2）php-fpm配置

```shell script
listen = 127.0.0.1:9000
# 或者下面这样
listen = /var/run/php-fpm.sock
```

> 注意，在使用 unix socket 方式连接时，由于 socket 文件本质上是一个文件，存在权限控制的问题，所以需要注意 nginx 进程的权限与 php-fpm 的权限问题，不然会提示无权限访问。（在各自的配置文件里设置用户）

通过以上的配置可以完成 `php-fpm` 与 `Nginx` 的通信

### 在应用中的选择
如果是在同一台服务器上运行的 `nginx` 和 `php-fpm`，且并发量不高（不超过 1000），选择 `unix socket`，以提高 `nginx` 和 `php-fpm` 的通信效率。

如果是面临高并发业务，则考虑选择使用更可靠的 tcp socket，以负载均衡、内核优化等运维手段维持效率。

> 文章内容部分来自网络，如有侵权，请联系我删除。
