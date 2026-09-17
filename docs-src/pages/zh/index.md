---
title: "cpp-httplib"
order: 0
---

[cpp-httplib](https://github.com/yhirose/cpp-httplib) 是一个用于 C++ 的 HTTP/HTTPS 库。只需复制单个头文件 [`httplib.h`](https://github.com/yhirose/cpp-httplib/raw/refs/tags/latest/httplib.h)，即可开始使用。

当你在 C++ 中需要快速搭建一个 HTTP 服务器或客户端时，你想要的就是一个开箱即用的东西。这正是我构建 cpp-httplib 的原因。只需几行代码，你就可以同时开始编写服务器和客户端。

API 采用基于 lambda 的设计，用起来自然顺手。只要有 C++11 或更高版本编译器的地方，它就能运行。Windows、macOS、Linux——用你手头现有的环境就好。

HTTPS 也不在话下。只需链接 OpenSSL 或 mbedTLS，服务器和客户端就都具备 TLS 支持。Content-Encoding（gzip、Brotli 等）、文件上传，以及其他实际开发中真正需要的功能全都包含在内。WebSocket 也同样支持。

在底层，它使用带线程池的阻塞 I/O。它并不是为处理海量并发连接而设计的。但对于 API 服务器、工具的嵌入式 HTTP 功能、测试用的 mock 服务器以及许多其他场景，它都能提供扎实的性能。

「今天的问题，今天解决。」这就是 cpp-httplib 追求的那种简单。

## 文档

- [cpp-httplib 向导](tour/) — 由浅入深的入门教程。新手请从这里开始
- [构建桌面 LLM 应用](llm-app/) — 使用 llama.cpp 一步步构建桌面应用的实战指南

## 敬请期待

- [Cookbook](cookbook/) — 按主题组织的配方集合。需要什么就跳到什么
