---
title: "S08. 返回压缩响应"
order: 27
status: "draft"
---

当客户端通过 `Accept-Encoding` 表明支持时，cpp-httplib 会自动压缩响应体。处理函数不需要做任何特殊处理。支持的编码有 gzip、Brotli 和 Zstd。

## 构建时配置

要启用压缩，在包含 `httplib.h` 之前定义相关的宏：

```cpp
#define CPPHTTPLIB_ZLIB_SUPPORT     // gzip
#define CPPHTTPLIB_BROTLI_SUPPORT   // brotli
#define CPPHTTPLIB_ZSTD_SUPPORT     // zstd
#include <httplib.h>
```

你还需要分别链接 `zlib`、`brotli` 和 `zstd`。只启用你需要的部分。

## 用法

```cpp
svr.Get("/api/data", [](const httplib::Request &req, httplib::Response &res) {
  std::string body = build_large_response();
  res.set_content(body, "application/json");
});
```

就这样。如果客户端发送了 `Accept-Encoding: gzip`，cpp-httplib 会自动用 gzip 压缩响应。`Content-Encoding: gzip` 和 `Vary: Accept-Encoding` 会为你加上。

## 编码优先级

当客户端接受多种编码时，cpp-httplib 按这个顺序选择（在构建时已启用的那些中）：Brotli → Zstd → gzip。你的代码不需要关心——你总是能拿到可用的最高效选项。

## 流式响应也会被压缩

通过 `set_chunked_content_provider()` 的流式响应会得到同样的自动压缩。

```cpp
svr.Get("/events", [](const httplib::Request &req, httplib::Response &res) {
  res.set_chunked_content_provider(
    "text/plain",
    [](size_t offset, httplib::DataSink &sink) {
      // ...
    });
});
```

## 静态文件需要显式启用

通过 `set_mount_point()` 或 `Response::set_file_content()` 原样提供的文件默认不会被压缩。用以下方式开启：

```cpp
svr.set_static_file_compression(true);
```

只有大小在某个范围内的文件会被压缩，而这个范围的两端都可以调整：

```cpp
svr.set_static_file_compression_min_length(512);
svr.set_static_file_compression_max_length(1024 * 1024);
```

下界默认为 1400 字节。一个已经能塞进单个 1500 字节 MTU 的响应，因为更小并不会投递得更快；而一个几字节的文件返回时会比传入时更大，因为 gzip 的头部和尾部开销超过了 deflate 省下的。

上界默认为 4MB，它存在的原因则不同：文件会在每次请求时被压缩，而压缩后的字节会一直驻留内存直到响应写完，因此峰值开销随同时请求数增长。它限制的是单个请求能花费的代价，并说明不了大文件能压缩得多好，所以当文件已知而流量未知时，提高它是合理的。

两个界都可以取 `0` 来关闭它，且各自都有一个编译期默认值（`CPPHTTPLIB_STATIC_FILE_COMPRESSION_MIN_LENGTH`、`CPPHTTPLIB_STATIC_FILE_COMPRESSION_MAX_LENGTH`）。

压缩后的响应会保留它的 `Content-Length`，所以 `HEAD` 报告的大小与 `GET` 会返回的一致。有两个细节要知道：Range 请求是基于未压缩的表示来应答的，而 `ETag` 会携带它所对应的编码，例如 `W/"...-gzip"`。

用 `set_content_provider()` 注册的 content provider 不在覆盖范围内。把其中一个穿过压缩器会将每次写入都暂在内部缓冲区直到填满，这会阻碍那些一块一块构建请求体的 provider。要压缩一个生成的请求体，使用 `set_chunked_content_provider()`。

> **提示：** 这个大小范围只涵盖静态文件。传给 `set_content()` 的请求体，只要客户端接受且 MIME 类型可压缩，无论多小都会被压缩，所以一个几字节的响应最终会比开始时更大。如果你要避免这一点，在处理函数里自行决定。

> 关于客户端侧的对应内容，参见 [C15. 启用压缩](../c15-compression)。
