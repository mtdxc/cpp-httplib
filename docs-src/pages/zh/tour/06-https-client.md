---
title: "HTTPS 客户端"
order: 6
---

上一章你配置好了 OpenSSL。现在把它用在 HTTPS 客户端上。你可以用第 2 章那个熟悉的 `httplib::Client`，只需向构造函数传入带 `https://` 协议的 URL。

## GET 请求

我们来访问一个真实的 HTTPS 站点。

```cpp
#define CPPHTTPLIB_OPENSSL_SUPPORT
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Client cli("https://nghttp2.org");

    auto res = cli.Get("/");
    if (res) {
        std::cout << res->status << std::endl;           // 200
        std::cout << res->body.substr(0, 100) << std::endl;  // First 100 chars of the HTML
    } else {
        std::cout << "Error: " << httplib::to_string(res.error()) << std::endl;
    }
}
```

第 2 章里你写的是 `httplib::Client cli("http://localhost:8080")`。需要改的只是把协议换成 `https://`。第 2 章学的所有 API——`Get()`、`Post()` 等——都完全一样地工作。

```sh
curl https://nghttp2.org/
```

## 指定端口

HTTPS 的默认端口是 443。如果你需要别的端口，把它写进 URL。

```cpp
httplib::Client cli("https://localhost:8443");
```

## CA 证书验证

通过 HTTPS 连接时，`httplib::Client` 默认会验证服务器证书。它只与那些证书由可信 CA（证书颁发机构）签发的服务器连接。

CA 证书会自动从 macOS 的钥匙串、Linux 的系统 CA 证书库、Windows 的证书库加载。大多数情况下无需额外配置。

### 指定 CA 证书文件

某些环境下可能找不到系统 CA 证书。这时用 `set_ca_cert_path()` 直接指定路径。

```cpp
httplib::Client cli("https://nghttp2.org");
cli.set_ca_cert_path("/etc/ssl/certs/ca-certificates.crt");

auto res = cli.Get("/");
```

```sh
curl --cacert /etc/ssl/certs/ca-certificates.crt https://nghttp2.org/
```

### 禁用证书验证

开发期间，你可能想连接一个自签名证书的服务器。可以为此禁用验证。

```cpp
httplib::Client cli("https://localhost:8443");
cli.enable_server_certificate_verification(false);

auto res = cli.Get("/");
```

```sh
curl -k https://localhost:8443/
```

切勿在生产环境中禁用它。这会让你暴露于中间人攻击。

## 跟随重定向

访问 HTTPS 站点时经常遇到重定向。例如从 `http://` 到 `https://`，或从裸域名到 `www`。

默认不跟随重定向。你可以在 `Location` 头里查看重定向目标。

```cpp
httplib::Client cli("https://nghttp2.org");

auto res = cli.Get("/httpbin/redirect/3");
if (res) {
    std::cout << res->status << std::endl;  // 302
    std::cout << res->get_header_value("Location") << std::endl;
}
```

```sh
curl https://nghttp2.org/httpbin/redirect/3
```

调用 `set_follow_location(true)` 就能自动跟随重定向，并获取最终响应。

```cpp
httplib::Client cli("https://nghttp2.org");
cli.set_follow_location(true);

auto res = cli.Get("/httpbin/redirect/3");
if (res) {
    std::cout << res->status << std::endl;  // 200 (the final response)
}
```

```sh
curl -L https://nghttp2.org/httpbin/redirect/3
```

## 下一步

现在你知道如何使用 HTTPS 客户端了。接下来搭建你自己的 HTTPS 服务器。我们从创建自签名证书开始。

**下一章：**[HTTPS 服务器](../07-https-server)
