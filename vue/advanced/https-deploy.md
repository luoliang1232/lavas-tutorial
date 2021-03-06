# HTTPS 环境部署

PWA 项目必须部署在 HTTPS 环境上才能够生效，主要是因为 Service Worker 只会在 HTTPS 环境下才能注册成功，我们不用担心在本地开发的时候 Service Worker 是否生效的问题，因为 Service Worker 在 `localhost` 和 `127.0.0.1` 的 host 下是能够注册成功的，这样可以确保我们在本地调试工作是能够顺利进行的。我们这里讲述的是如何部署线上的 https 环境来确保我们的 PWA 应用成功运行。

## 什么是 HTTPS

我们都知道 Web App 的运行都是建立在网络应用层 HTTP 协议的，HTTP 协议能够进行客户端和服务器之间的请求和返回。但是这个过程是明文传输的，当请求被抓包后传输内容很容易被篡改，这对用户的安全性来说是极其严重的威胁。PWA 应用出于安全性的考虑要求项目必须部署在 HTTPS 环境。

那么 HTTPS 是什么呢？

HTTPS 是将 HTTP 置于 SSL/TLS 之上，其效果是加密 HTTP 流量( traffic )，包括请求的 URL、结果页面、cookies、媒体资源和其他通过 HTTP 传输的内容。企图干扰 HTTPS 连接的人既无法监听流量，也无法更改其内容。除了加密，远程服务器的身份也要进行验证：毕竟，如果你无法确定连接的另一端是谁，加密连接也就没什么意义了。这些措施将使拦截流量变得极其困难。虽然攻击者仍有可能知道用户正在访问哪个网站，但他所能知道的也就仅限于此了。

## 获取证书

如果需要部署 https 环境，我们就需要从 CA (Certificate Authority) 获取一个 DV (Domain Validation) 证书。

获取的证书如果是永久有效的一般都是需要付费的，当然也有便宜的和免费的，视具体需求而定。如果是个人用户，可以选择便宜甚至免费的 DV 证书；如果是有更高要求的站点，可以选择增值服务更加完善的服务商。

这里我们重点介绍的是一个免费的证书 CA：[Let's Encrypt](https://letsencrypt.org/)

特点：免费，快捷，支持多域名（不是通配符），三条命令即时签署 + 导出证书。
缺点：存在有效期限制，到期需续签。

具体的获取证书的方法可以参考：[https://foofish.net/https-free-for-lets-encrypt.html](https://foofish.net/https-free-for-lets-encrypt.html)

## Nginx 部署 HTTPS

参照：[https://foofish.net/https-free-for-lets-encrypt.html](https://foofish.net/https-free-for-lets-encrypt.html)
在成功获取证书之后，通常我们只需要修改 Nginx 中有关证书的配置并 reload 服务即可：

```nginx
server {
    listen 443;
    server_name yourwebsit.com, www.yourwebsit.net;
    ssl on;
    ssl_certificate /path/to/chained.pem;
    ssl_certificate_key /path/to/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    ssl_prefer_server_ciphers on;

    ...the rest of your config
}
```

## Node.js server 部署 HTTPS

在 Node.js 中我们如何部署 HTTPS 环境呢？假设我们使用 `Let's Encrypt` 已经拿到了 DV 证书。

* private.pe: 私钥文件
* file.crt: 证书文件

如果使用 express 作为 server 服务器：

```js
var app = require('express')();
var fs = require('fs');
var http = require('http');
var https = require('https');
var privateKey  = fs.readFileSync('/path/to/private.pem', 'utf8');
var certificate = fs.readFileSync('/path/to/file.crt', 'utf8');
var credentials = {key: privateKey, cert: certificate};

var httpServer = http.createServer(app);
var httpsServer = https.createServer(credentials, app);
var PORT = 8080;
var SSLPORT = 8081;

httpServer.listen(PORT, function () {
    console.log('HTTP Server is running on: http://localhost:%s', PORT);
});
httpsServer.listen(SSLPORT, function () {
    console.log('HTTPS Server is running on: https://localhost:%s', SSLPORT);
});

// Welcome
app.get('/', function (req, res) {
    if (req.protocol === 'https') {
        res.status(200).send('Welcome to Safety Land!');
    }
    else {
        res.status(200).send('Welcome!');
    }
});
```