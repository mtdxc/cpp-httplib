---
title: "S16. 检测客户端何时断开"
order: 35
status: "draft"
---

在一个长时间运行的响应期间，客户端可能关闭连接。继续做没人等待的工作没有意义。在 cpp-httplib 中，检查 `req.is_connection_closed()`。

## 基本用法

```cpp
svr.Get("/long-task", [](const httplib::Request &req, httplib::Response &res) {
  for (int i = 0; i < 1000; ++i) {
    if (req.is_connection_closed()) {
      std::cout << "client disconnected" << std::endl;
      return;
    }

    do_heavy_work(i);
  }

  res.set_content("done", "text/plain");
});
```

`is_connection_closed` 是一个 `std::function<bool()>`，所以用 `()` 调用它。当客户端离开时它返回 `true`。

## 与流式响应一起使用

同样的检查在 `set_chunked_content_provider()` 内部也适用。按引用捕获请求。

```cpp
svr.Get("/events", [](const httplib::Request &req, httplib::Response &res) {
  res.set_chunked_content_provider(
    "text/event-stream",
    [&req](size_t offset, httplib::DataSink &sink) {
      if (req.is_connection_closed()) {
        sink.done();
        return true;
      }

      auto event = generate_next_event();
      sink.write(event.data(), event.size());
      return true;
    });
});
```

当你检测到断开时，调用 `sink.done()` 以阻止 provider 再次被调用。

## 应该多久检查一次？

调用本身很廉价，但在紧密的内层循环里调用它不会增加多少价值。在**可以安全中断的边界**上检查——生成一块数据后、一次数据库查询后等。

> **警告：** `is_connection_closed()` 不保证能即时反映真实情况。由于 TCP 的工作方式，有时你只有在尝试发送时才会注意到断开。不要指望像素级的实时检测——把它当作“我们迟早会注意到”。
