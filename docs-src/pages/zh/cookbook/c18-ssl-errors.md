---
title: "C18. 处理 SSL 错误"
order: 18
status: "draft"
---

当一个 HTTPS 请求失败时，`res.error()` 会返回像 `Error::SSLConnection` 或 `Error::SSLServerVerification` 这样的值。有时这不足以定位原因。这时 `Result::ssl_error()` 和 `Result::ssl_backend_error()` 就能帮忙。

## 获取 SSL 错误详情

```cpp
httplib::Client cli("https://api.example.com");
auto res = cli.Get("/");

if (!res) {
  auto err = res.error();
  std::cerr << "error: " << httplib::to_string(err) << std::endl;

  if (err == httplib::Error::SSLConnection ||
      err == httplib::Error::SSLServerVerification) {
    std::cerr << "ssl_error: " << res.ssl_error() << std::endl;
    std::cerr << "ssl_backend_error: " << res.ssl_backend_error() << std::endl;
  }
}
```

`ssl_error()` 返回来自 SSL 库的错误码（例如 OpenSSL 的 `SSL_get_error()`）。`ssl_backend_error()` 给你后端的更详细错误值——对 OpenSSL 而言，就是 `ERR_get_error()`。

## 将 OpenSSL 错误格式化为字符串

当你有一个来自 `ssl_backend_error()` 的值时，把它传给 OpenSSL 的 `ERR_error_string()` 以获得可读的消息。

```cpp
#include <openssl/err.h>

if (res.ssl_backend_error() != 0) {
  char buf[256];
  ERR_error_string_n(res.ssl_backend_error(), buf, sizeof(buf));
  std::cerr << "openssl: " << buf << std::endl;
}
```

## 常见原因

| 症状 | 常见原因 |
| --- | --- |
| `SSLServerVerification` | CA 证书路径未配置，或证书为自签名 |
| `SSLServerHostnameVerification` | 证书的 CN/SAN 与主机不匹配 |
| `SSLConnection` | TLS 版本不匹配，没有共享的加密套件 |

> 要更改证书验证设置，参见 [T02. 控制 SSL 证书验证](../t02-cert-verification)。
