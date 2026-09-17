---
title: "下一步"
order: 9
---

恭喜你完成了向导！现在你已经掌握了 cpp-httplib 的坚实基硜。但还有很多值得探索的东西。以下是向导中未涵盖功能的概览，按类别整理。

## 流式 API

处理 LLM 流式响应或下载大文件时，你不想把整个响应加载到内存。用 `stream::Get()` 逐块处理数据。

```cpp
httplib::Client cli("http://localhost:11434");

auto result = httplib::stream::Get(cli, "/api/generate");

if (result) {
    while (result.next()) {
        std::cout.write(result.data(), result.size());
    }
}
```

你也可以给 `Get()` 传入一个 `content_receiver` 回调。这种方式兼容 Keep-Alive。

```cpp
httplib::Client cli("http://localhost:8080");

cli.Get("/stream", [](const char *data, size_t len) {
    std::cout.write(data, len);
    return true;
});
```

服务器端，你有 `set_content_provider()` 和 `set_chunked_content_provider()`。知道大小时用前者，不知道时用后者。

```cpp
// With known size (sets Content-Length)
svr.Get("/file", [](const auto &, auto &res) {
    auto size = get_file_size("large.bin");
    res.set_content_provider(size, "application/octet-stream",
        [](size_t offset, size_t length, httplib::DataSink &sink) {
            // Send 'length' bytes starting from 'offset'
            return true;
        });
});

// Unknown size (Chunked Transfer Encoding)
svr.Get("/stream", [](const auto &, auto &res) {
    res.set_chunked_content_provider("text/plain",
        [](size_t offset, httplib::DataSink &sink) {
            sink.write("chunk\n", 6);
            return true;  // Return false to finish
        });
});
```

上传大文件时，`make_file_provider()` 很好用。它会流式传输文件，而不是全部加载到内存。

```cpp
httplib::Client cli("http://localhost:8080");

auto res = cli.Post("/upload", {}, {}, {
    httplib::make_file_provider("file", "/path/to/large-file.zip")
});
```

## Server-Sent Events（SSE）

我们也提供 SSE 客户端。它支持自动重连和通过 `Last-Event-ID` 续传。

```cpp
httplib::Client cli("http://localhost:8080");
httplib::sse::SSEClient sse(cli, "/events");

sse.on_message([](const httplib::sse::SSEMessage &msg) {
    std::cout << msg.event << ": " << msg.data << std::endl;
});

sse.start();  // Blocking, with auto-reconnection
```

也可以为每种事件类型分设处理器。

```cpp
sse.on_event("update", [](const httplib::sse::SSEMessage &msg) {
    // Only handles "update" events
});
```

## 认证

客户端内置了 Basic 认证、Bearer Token 认证和 Digest 认证的辅助方法。

```cpp
httplib::Client cli("https://api.example.com");
cli.set_basic_auth("user", "password");
cli.set_bearer_token_auth("my-token");
```

## 压缩

支持 gzip、Brotli 和 Zstandard 的压缩与解压。编译时定义对应的宏。

| 方式 | 宏 |
| -- | -- |
| gzip | `CPPHTTPLIB_ZLIB_SUPPORT` |
| Brotli | `CPPHTTPLIB_BROTLI_SUPPORT` |
| Zstandard | `CPPHTTPLIB_ZSTD_SUPPORT` |

```cpp
httplib::Client cli("https://example.com");
cli.set_compress(true);    // Compress request body
cli.set_decompress(true);  // Decompress response body
```

## 代理

可以通过 HTTP 代理连接。

```cpp
httplib::Client cli("https://example.com");
cli.set_proxy("proxy.example.com", 8080);
cli.set_proxy_basic_auth("user", "password");
```

## 超时

可以分别设置连接、读取和写入超时。

```cpp
httplib::Client cli("https://example.com");
cli.set_connection_timeout(5, 0);  // 5 seconds
cli.set_read_timeout(10, 0);       // 10 seconds
cli.set_write_timeout(10, 0);      // 10 seconds
```

## Keep-Alive

如果要向同一个服务器发多个请求，启用 Keep-Alive。它会复用 TCP 连接，效率高得多。

```cpp
httplib::Client cli("https://example.com");
cli.set_keep_alive(true);
```

## 服务器中间件

你可以在处理器运行前后hook请求处理。

```cpp
svr.set_pre_routing_handler([](const auto &req, auto &res) {
    // Runs before every request
    return httplib::Server::HandlerResponse::Unhandled;  // Continue to normal routing
});

svr.set_post_routing_handler([](const auto &req, auto &res) {
    // Runs after the response is sent
    res.set_header("X-Server", "cpp-httplib");
});
```

用 `res.user_data` 把数据从中间件传递给处理器。这对共享已解码的认证 token 之类的东西很有用。

```cpp
svr.set_pre_routing_handler([](const auto &req, auto &res) {
    res.user_data.set("auth_user", std::string("alice"));
    return httplib::Server::HandlerResponse::Unhandled;
});

svr.Get("/me", [](const auto &req, auto &res) {
    auto *user = res.user_data.get<std::string>("auth_user");
    res.set_content("Hello, " + *user, "text/plain");
});
```

你也可以自定义错误和异常处理器。

```cpp
svr.set_error_handler([](const auto &req, auto &res) {
    res.set_content("Custom Error Page", "text/html");
});

svr.set_exception_handler([](const auto &req, auto &res, std::exception_ptr ep) {
    res.status = 500;
    res.set_content("Internal Server Error", "text/plain");
});
```

## 日志

可以在服务器和客户端都设置日志器。

```cpp
svr.set_logger([](const auto &req, const auto &res) {
    std::cout << req.method << " " << req.path << " " << res.status << std::endl;
});
```

## Unix 域套接字

除了 TCP，我们还支持 Unix 域套接字。可用于同一台机器上的进程间通信。

```cpp
// Server
httplib::Server svr;
svr.set_address_family(AF_UNIX);
svr.listen("/tmp/httplib.sock", 0);
```

```cpp
// Client
httplib::Client cli("http://localhost");
cli.set_address_family(AF_UNIX);
cli.set_hostname_addr_map({{"localhost", "/tmp/httplib.sock"}});

auto res = cli.Get("/");
```

## 了解更多

想深入研究？看看这些资源。

- Cookbook — 常见用例的配方集合
- [README](https://github.com/yhirose/cpp-httplib/blob/master/README.md) — 完整的 API 参考
- [README-sse](https://github.com/yhirose/cpp-httplib/blob/master/README-sse.md) — 如何使用 Server-Sent Events
- [README-stream](https://github.com/yhirose/cpp-httplib/blob/master/README-stream.md) — 如何使用流式 API
- [README-websocket](https://github.com/yhirose/cpp-httplib/blob/master/README-websocket.md) — 如何使用 WebSocket 服务器
