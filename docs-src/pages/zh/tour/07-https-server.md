---
title: "HTTPS 服务器"
order: 7
---

上一章你用了 HTTPS 客户端。现在来搭建你自己的 HTTPS 服务器。只需把第 3 章的 `httplib::Server` 换成 `httplib::SSLServer`。

不过 TLS 服务器需要服务器证书和私钥。我们先把它们准备好。

## 创建自签名证书

开发和测试时，自签名证书就完全够用。你可以用一条 OpenSSL 命令快速生成。

```sh
openssl req -x509 -noenc -keyout key.pem -out cert.pem -subj /CN=localhost
```

这会生成两个文件：

- **`cert.pem`** — 服务器证书
- **`key.pem`** — 私钥

## 一个极简 HTTPS 服务器

有了证书，我们来实现服务器。

```cpp
#define CPPHTTPLIB_OPENSSL_SUPPORT
#include "httplib.h"
#include <iostream>

int main() {
    httplib::SSLServer svr("cert.pem", "key.pem");

    svr.Get("/", [](const auto &, auto &res) {
        res.set_content("Hello, HTTPS!", "text/plain");
    });

    std::cout << "Listening on https://localhost:8443" << std::endl;
    svr.listen("0.0.0.0", 8443);
}
```

只需把证书和私钥路径传给 `httplib::SSLServer` 构造函数。路由 API 与第 3 章的 `httplib::Server` 完全相同。

编译并启动它。

## 试一试

服务器运行后，用 `curl` 访问试试。因为我们用的是自签名证书，加上 `-k` 选项跳过证书验证。

```sh
curl -k https://localhost:8443/
# Hello, HTTPS!
```

如果在浏览器打开 `https://localhost:8443`，你会看到「此连接不安全」的警告。自签名证书下这是意料之中的。继续访问即可。

## 从客户端连接

我们用上一章的 `httplib::Client` 来连接。连接自签名证书的服务器有两种方式。

### 方式一：禁用证书验证

这是开发时快捷简单的做法。

```cpp
#define CPPHTTPLIB_OPENSSL_SUPPORT
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Client cli("https://localhost:8443");
    cli.enable_server_certificate_verification(false);

    auto res = cli.Get("/");
    if (res) {
        std::cout << res->body << std::endl;  // Hello, HTTPS!
    }
}
```

### 方式二：把自签名证书指定为 CA 证书

这是更安全的做法。你告诉客户端把 `cert.pem` 当作 CA 证书信任。

```cpp
#define CPPHTTPLIB_OPENSSL_SUPPORT
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Client cli("https://localhost:8443");
    cli.set_ca_cert_path("cert.pem");

    auto res = cli.Get("/");
    if (res) {
        std::cout << res->body << std::endl;  // Hello, HTTPS!
    }
}
```

这样，只允许连接到持有那个特定证书的服务器，防止被仿冒。即使在测试环境，也尽量使用这种方式。

## 对比 Server 和 SSLServer

你在第 3 章学的 `httplib::Server` API，在 `httplib::SSLServer` 上完全一样地工作。唯一的区别是构造函数。

| | `httplib::Server` | `httplib::SSLServer` |
| -- | ------------------ | -------------------- |
| 构造函数 | 无参数 | 证书和私钥路径 |
| 协议 | HTTP | HTTPS |
| 端口（惯例） | 8080 | 8443 |
| 路由 | 相同 | 相同 |

把 HTTP 服务器换成 HTTPS，只需改构造函数。

## 下一步

你的 HTTPS 服务器已经跑起来了。HTTP/HTTPS 的客户端和服务器基础你都已掌握。

接下来看看 cpp-httplib 新增的 WebSocket 支持。

**下一章：**[WebSocket](../08-websocket)
