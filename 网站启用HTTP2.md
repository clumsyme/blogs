[imliyan.com](https://imliyan.com)终于启用了HTTP/2了。

## 为什么之前没支持

好吧，其实HTTP/2在本站是一直开启的，只是由于Google在去年[不再支持被OpenSSL 1.0.2以下版本广泛支持的`NPN`](https://bugs.chromium.org/p/chromium/issues/detail?id=527066)，而转向了`ALPN`，但`ALPN`则最低需要`OpenSSL 1.0.2`版本，而目前的linux服务器，除了`Ubuntu 16.04 LTS`以及`Debian 9.0`，其他OpenSSL版本都低于1.0.2。由于OpenSSL是系统服务，直接升级OpenSSL版本可能导致不可预知的系统问题。

## 不升级OpenSSL怎么实现

1. 最直接的： 升级系统到`Ubuntu 16.04 LTS`或`Debian 9.0`。

2. 使用`OpenSSL 1.0.2`之后的版本从源码重新编译一个Nginx版本。（具体可参考[Supporting HTTP/2 for Website Visitors - What Can You Do?](https://www.nginx.com/blog/supporting-http2-google-chrome-users/)）

## HTTP/2 的好处

升级HTTP/2，一大特性就是单个TCP连接并行处理多个请求的多路复用特性，有效提升网站速度。现在升级完，试了下，好像也没有快很多啊。 :-(

不过在国内连国外的服务器能快到那里呢。

# ：）