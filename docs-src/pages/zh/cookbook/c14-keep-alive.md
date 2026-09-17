---
title: "C14. 理解连接复用与 Keep-Alive"
order: 14
status: "draft"
---

当你通过同一个 `httplib::Client` 实例发送多个请求时，TCP 连接会被自动复用。HTTP/1.1 的 Keep-Alive 替你完成了工作——你无需在每次调用上都付出 TCP 和 TLS 握手的开销。

## 连接会被自动复用

```cpp
httplib::Client cli("https://api.example.com");

auto res1 = cli.Get("/users/1");
auto res2 = cli.Get("/users/2"); // reuses the same connection
auto res3 = cli.Get("/users/3"); // reuses the same connection
```

无需特殊配置。只要保持住 `cli`——在内部，socket 会在多次调用间保持打开。在 HTTPS 上效果尤为明显，因为 TLS 握手开销很大。

## 显式禁用 Keep-Alive

要强制每次都新建一个连接，调用 `set_keep_alive(false)`。主要用于测试。

```cpp
cli.set_keep_alive(false);
```

对于正常使用，保持开启（默认值）即可。

## 不要为每个请求创建一个 `Client`

如果你在循环内部创建 `Client` 并让它在每轮迭代离开作用域，你就失去了复用的好处。在循环外创建实例。

```cpp
// Bad: a new connection every iteration
for (auto id : ids) {
  httplib::Client cli("https://api.example.com");
  cli.Get("/users/" + id);
}

// Good: the connection is reused
httplib::Client cli("https://api.example.com");
for (auto id : ids) {
  cli.Get("/users/" + id);
}
```

## 并发请求

如果你想从多个线程并行发送请求，给每个线程自己的 `Client` 实例。单个 `Client` 使用单条 TCP 连接，因此向同一个实例并发发送请求最终也会把它们串行化。

> **提示：** 如果服务端在其 Keep-Alive 超时后关闭了连接，cpp-httplib 会透明地重连并重试。你无需在应用代码中处理这一点。
