---
title: "T01. 在 OpenSSL、mbedTLS 与 wolfSSL 之间选择"
order: 43
status: "draft"
---

cpp-httplib 不自带 TLS 实现——它使用三个后端之一，你在构建时通过一个宏选择。

| 后端 | 宏 | 特点 |
| --- | --- | --- |
| OpenSSL | `CPPHTTPLIB_OPENSSL_SUPPORT` | 使用最广，功能集最丰富 |
| mbedTLS | `CPPHTTPLIB_MBEDTLS_SUPPORT` | 轻量，面向嵌入式 |
| wolfSSL | `CPPHTTPLIB_WOLFSSL_SUPPORT` | 对嵌入式友好，有商业支持 |

## 构建时选择

在包含 `httplib.h` 之前，为你选定的后端定义宏：

```cpp
#define CPPHTTPLIB_OPENSSL_SUPPORT
#include <httplib.h>
```

你还需要链接后端的库（`libssl`、`libcrypto`、`libmbedtls`、`libwolfssl` 等）。

## 选哪个

**拿不定主意时，选 OpenSSL**  
它功能最多、文档最好。对于常规的服务端使用或 Linux 桌面应用，从这里开始——你可能不需要别的。

**要缩小二进制体积或面向嵌入式**  
mbedTLS 或 wolfSSL 更合适。它们比 OpenSSL 紧凑得多，能跑在内存受限的设备上。

**当你需要商业支持时**  
wolfSSL 提供商业许可和支持。如果你在商业产品中交付，值得考虑。

## 支持多个后端

常规做法是把每个后端当作一个构建变体，用不同的宏重新编译同一份源码。cpp-httplib 平滑了大部分 API 差异，但后端并非 100% 相同——务必测试。

## 在所有后端上都适用的 API

证书验证控制、搭建一个 SSLServer、读取对端证书——这些在所有后端上都共享同样的 API：

- [T02. 控制 SSL 证书验证](../t02-cert-verification)
- [T03. 启动一个 SSL/TLS 服务端](../t03-ssl-server)
- [T05. 在服务端访问对端证书](../t05-peer-cert)

> **提示：** 在使用 OpenSSL 系列后端的 macOS 上，cpp-httplib 会自动从系统钥匙串加载根证书（通过 `CPPHTTPLIB_USE_CERTS_FROM_MACOSX_KEYCHAIN`，默认开启）。要禁用这一点，定义 `CPPHTTPLIB_DISABLE_MACOSX_AUTOMATIC_ROOT_CERTIFICATES`。
