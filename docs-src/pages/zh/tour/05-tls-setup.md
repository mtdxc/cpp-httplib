---
title: "TLS 设置"
order: 5
---

到目前为止我们用的都是明文 HTTP，但在真实世界里 HTTPS 才是常态。要让 cpp-httplib 使用 HTTPS，你需要一个 TLS 库。

本教程使用 OpenSSL。它是最常用的选择，网上也能找到大量资料。

## 安装 OpenSSL

按你的操作系统安装。

| 操作系统 | 安装方式 |
| -- | -------------- |
| macOS | [Homebrew](https://brew.sh/)（`brew install openssl`） |
| Ubuntu / Debian | `sudo apt install libssl-dev` |
| Windows | [vcpkg](https://vcpkg.io/)（`vcpkg install openssl`） |

## 编译选项

要启用 TLS，编译时需定义 `CPPHTTPLIB_OPENSSL_SUPPORT` 宏。与前面的章节相比，需要多几个选项。

```sh
# macOS (Homebrew)
clang++ -std=c++17 -DCPPHTTPLIB_OPENSSL_SUPPORT \
    -I$(brew --prefix openssl)/include \
    -L$(brew --prefix openssl)/lib \
    -lssl -lcrypto \
    -framework CoreFoundation -framework Security \
    -o server server.cpp

# Linux
clang++ -std=c++17 -pthread -DCPPHTTPLIB_OPENSSL_SUPPORT \
    -lssl -lcrypto \
    -o server server.cpp

# Windows (Developer Command Prompt)
cl /EHsc /std:c++17 /DCPPHTTPLIB_OPENSSL_SUPPORT server.cpp libssl.lib libcrypto.lib
```

来看看每个选项的作用。

- **`-DCPPHTTPLIB_OPENSSL_SUPPORT`** — 定义启用 TLS 支持的宏
- **`-lssl -lcrypto`** — 链接 OpenSSL 库
- **`-I` / `-L`**（仅 macOS）— 指向 Homebrew 的 OpenSSL 路径
- **`-framework CoreFoundation -framework Security`**（仅 macOS）— 从钥匙串自动加载系统证书时需要

## 验证环境

确认一切都正常。下面是一个简单程序，把 HTTPS URL 传给 `httplib::Client`。

```cpp
#define CPPHTTPLIB_OPENSSL_SUPPORT
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Client cli("https://www.google.com");

    auto res = cli.Get("/");
    if (res) {
        std::cout << "Status: " << res->status << std::endl;
    } else {
        std::cout << "Error: " << httplib::to_string(res.error()) << std::endl;
    }
}
```

编译并运行。如果看到 `Status: 200`，环境就绪。

## 其他 TLS 后端

除 OpenSSL 外，cpp-httplib 还支持 Mbed TLS 和 wolfSSL。只需更换宏定义和链接的库就能在它们之间切换。

| 后端 | 宏 | 链接的库 |
| :--- | :--- | :--- |
| OpenSSL | `CPPHTTPLIB_OPENSSL_SUPPORT` | `libssl`、`libcrypto` |
| Mbed TLS | `CPPHTTPLIB_MBEDTLS_SUPPORT` | `libmbedtls`、`libmbedx509`、`libmbedcrypto` |
| wolfSSL | `CPPHTTPLIB_WOLFSSL_SUPPORT` | `libwolfssl` |

支持Mbed TLS 2.x、3.x、4.x 且会自动检测。注意 Mbed TLS 4.x 把 `libmbedcrypto` 改名为 `libtfpsacrypto`，需改链接它。

本教程以 OpenSSL 为例，但无论你选哪个后端，API 都是一样的。

## 下一步

TLS 已就绪。接下来向 HTTPS 站点发送一个请求。

**下一章：**[HTTPS 客户端](../06-https-client)
