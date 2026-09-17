---
title: "E04. 在客户端接收 SSE"
order: 51
status: "draft"
---

cpp-httplib 自带一个专用的 `sse::SSEClient` 类。它为你处理自动重连、按事件名称分派以及 `Last-Event-ID` 跟踪——因此接收 SSE 毫不费力。

## 基本用法

```cpp
#include <httplib.h>

httplib::Client cli("http://localhost:8080");
httplib::sse::SSEClient sse(cli, "/events");

sse.on_message([](const httplib::sse::SSEMessage &msg) {
  std::cout << "data: " << msg.data << std::endl;
});

sse.start(); // blocking
```

用一个 `Client` 和一个路径构建 `SSEClient`，用 `on_message()` 注册一个回调，然后调用 `start()`。事件循环会启动，如果连接中断会自动重连。

## 按事件名称分派

当服务端发送带 `event:` 字段的事件时，通过 `on_event()` 按名称注册处理函数。

```cpp
sse.on_event("message", [](const auto &msg) {
  std::cout << "chat: " << msg.data << std::endl;
});

sse.on_event("join", [](const auto &msg) {
  std::cout << msg.data << " joined" << std::endl;
});

sse.on_event("leave", [](const auto &msg) {
  std::cout << msg.data << " left" << std::endl;
});
```

`on_message()` 作为无名事件（默认的 `message` 类型）的通用回退。

## 连接生命周期与错误

```cpp
sse.on_open([] {
  std::cout << "connected" << std::endl;
});

sse.on_error([](httplib::Error err) {
  std::cerr << "error: " << httplib::to_string(err) << std::endl;
});
```

挂钩连接打开和错误事件。即使错误处理器触发了，`SSEClient` 也会在后台继续尝试重连。

## 异步运行

如果你不想阻塞主线程，使用 `start_async()`。

```cpp
sse.start_async();

// main thread continues to do other things
do_other_work();

// when you're done, stop it
sse.stop();
```

`start_async()` 会派生一个后台线程来运行事件循环。用 `stop()` 干净地关闭它。

## 配置重连

你可以调整重连间隔和最大重试次数。

```cpp
sse.set_reconnect_interval(5000);    // 5 seconds
sse.set_max_reconnect_attempts(10);  // up to 10 (0 = unlimited)
```

如果服务端发送了一个 `retry:` 字段，它优先。

## 自动 Last-Event-ID

`SSEClient` 会在内部跟踪每个收到的事件的 `id`，并在重连时将其作为 `Last-Event-ID` 发回。只要服务端发送带 `id:` 的事件，这一切都会自动工作。

```cpp
std::cout << "last id: " << sse.last_event_id() << std::endl;
```

用 `last_event_id()` 读取当前值。

> **提示：** `SSEClient::start()` 会阻塞，对于一个一次性的命令行工具来说这没问题。对于 GUI 应用或嵌入到服务端的情况，`start_async()` + `stop()` 这对才是常规写法。

> 关于服务端侧，参见 [E01. 实现一个 SSE 服务端](../e01-sse-server)。
