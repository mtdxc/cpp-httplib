---
title: "T02. 控制 SSL 证书验证"
order: 44
status: "draft"
---

默认情况下，HTTPS 客户端会验证服务端证书——它使用操作系统的根证书存储来检查证书链和主机名。下面是改变这一行为的 API。

## 指定自定义 CA 证书

当连接一个证书由内部 CA 签名的服务端时，使用 `set_ca_cert_path()`。

```cpp
httplib::Client cli("https://internal.example.com");
cli.set_ca_cert_path("/etc/ssl/certs/internal-ca.pem");

auto res = cli.Get("/");
```

第一个参数是 CA 证书文件；第二个是可选的 CA 目录。使用 OpenSSL 后端时，你还可以通过 `set_ca_cert_store()` 直接传入一个 `X509_STORE*`。

## 禁用证书验证（不推荐）

对于开发服务端或自签名证书，你可以完全跳过验证。

```cpp
httplib::Client cli("https://self-signed.example.com");
cli.enable_server_certificate_verification(false);

auto res = cli.Get("/");
```

这就是禁用链验证所需的全部。

> **警告：** 禁用证书验证会移除针对中间人攻击的防护。**绝不要在生产环境中这样做。** 如果你在开发/测试之外发现自己需要它，停下来，确保你没做错什么。

## 仅禁用主机名验证

有一个中间选项：验证证书链，但跳过主机名检查。当你需要访问一个证书 CN/SAN 与请求主机名不匹配的服务端时很有用。

```cpp
cli.enable_server_hostname_verification(false);
```

证书本身仍会被验证，因此这比完全禁用验证更安全——但仍不推荐用于生产。

## 原样使用操作系统证书存储

在大多数 Linux 发行版上，根证书位于像 `/etc/ssl/certs/ca-certificates.crt` 这样的单个文件中。cpp-httplib 在启动时读取操作系统的默认存储，所以对于大多数服务端你不需要配置任何东西。

> 同样的 API 在 mbedTLS 和 wolfSSL 后端上也适用。关于在后端之间选择，参见 [T01. 在 OpenSSL、mbedTLS 与 wolfSSL 之间选择](../t01-tls-backends)。

> 关于诊断失败的详情，参见 [C18. 处理 SSL 错误](../c18-ssl-errors)。

> 关于 WebSocket 客户端（`wss://`）上的 TLS 配置，参见 [W05. 为 wss:// 连接配置 TLS](../w05-websocket-tls)。
