---
title: "E01. 实现一个 SSE 服务端"
order: 48
status: "draft"
---

Server-Sent Events（SSE）是一种简单的协议，用于从服务端向客户端单向推送事件。连接保持打开，服务端可以在任意时刻发送数据。它比 WebSocket 更轻量，并且完全包含在 HTTP 之内——一个不错的组合。

cpp-httplib 没有专用的 SSE 服务端 API，但你可以用 `set_chunked_content_provider()` 和 `text/event-stream` 来实现一个。

## 基本的 SSE 服务端

```cpp
svr.Get("/events", [](const httplib::Request &req, httplib::Response &res) {
  res.set_chunked_content_provider(
    "text/event-stream",
    [](size_t offset, httplib::DataSink &sink) {
      std::string message = "data: hello\n\n";
      sink.write(message.data(), message.size());
      std::this_thread::sleep_for(std::chrono::seconds(1));
      return true;
    });
});
```

这里有三点很重要：

1. Content-Type 是 `text/event-stream`
2. 消息遵循 `data: <content>\n\n` 格式（双换行分隔事件）
3. 每次 `sink.write()` 都会把数据交付给客户端

只要连接存活，provider lambda 就会持续被调用。

## 一个连续流

下面是一个每秒发送当前时间的简单示例。

```cpp
svr.Get("/time", [](const httplib::Request &req, httplib::Response &res) {
  res.set_chunked_content_provider(
    "text/event-stream",
    [&req](size_t offset, httplib::DataSink &sink) {
      if (req.is_connection_closed()) {
        sink.done();
        return true;
      }

      auto now = std::chrono::system_clock::now();
      auto t = std::chrono::system_clock::to_time_t(now);
      std::string msg = "data: " + std::string(std::ctime(&t)) + "\n";
      sink.write(msg.data(), msg.size());

      std::this_thread::sleep_for(std::chrono::seconds(1));
      return true;
    });
});
```

当客户端断开时，调用 `sink.done()` 以停止。详情见 [S16. 检测客户端断开](../s16-disconnect)。

## 通过注释行发送心跳

以 `:` 开头的行是 SSE 注释——客户端会忽略它们，但它们会**保持连接存活**。对于防止代理和负载均衡器关闭空闲连接很有用。

```cpp
// heartbeat every 30 seconds
if (tick_count % 30 == 0) {
  std::string ping = ": ping\n\n";
  sink.write(ping.data(), ping.size());
}
```

## 与线程池的关系

SSE 连接会保持打开，所以每个客户端都占用一个工作线程。对于大量并发连接，启用线程池的动态扩容。

```cpp
svr.new_task_queue = [] {
  return new httplib::ThreadPool(8, 128);
};
```

参见 [S21. 配置线程池](../s21-thread-pool)。

> **提示：** 当 `data:` 包含换行时，把它拆成多个 `data:` 行——每行一个。这就是 SSE 规范要求多行数据的传输方式。

> 关于事件名称，参见 [E02. 在 SSE 中使用命名事件](../e02-sse-event-names)。关于客户端侧，参见 [E04. 在客户端接收 SSE](../e04-sse-client)。
