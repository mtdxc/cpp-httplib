---
title: "C15. 启用压缩"
order: 15
status: "draft"
---

cpp-httplib 支持发送时压缩、接收时解压。你只需启用 zlib 或 Brotli 来构建它。

## 构建时配置

要使用压缩，在包含 `httplib.h` 之前定义这些宏：

```cpp
#define CPPHTTPLIB_ZLIB_SUPPORT    // gzip / deflate
#define CPPHTTPLIB_BROTLI_SUPPORT  // brotli
#include <httplib.h>
```

你还需要链接 `zlib` 或 `brotli`。

## 压缩请求体

```cpp
httplib::Client cli("https://api.example.com");
cli.set_compress(true);

std::string big_payload = build_payload();
auto res = cli.Post("/api/data", big_payload, "application/json");
```

启用 `set_compress(true)` 后，POST 或 PUT 请求的请求体会在发送前被 gzip 压缩。服务端也需要能处理压缩的请求体。

## 解压响应

```cpp
httplib::Client cli("https://api.example.com");
cli.set_decompress(true); // on by default

auto res = cli.Get("/api/data");
std::cout << res->body << std::endl;
```

启用 `set_decompress(true)` 后，客户端会自动解压以 `Content-Encoding: gzip` 或类似方式到达的响应。`res->body` 包含解压后的数据。

它默认开启，所以通常你什么都不用做。只有当你想要原始压缩字节时才将其设为 `false`。

> **警告：** 如果你在构建时没有启用 `CPPHTTPLIB_ZLIB_SUPPORT`，调用 `set_compress()` 或 `set_decompress()` 不会有任何作用。如果压缩没生效，先检查宏定义。
