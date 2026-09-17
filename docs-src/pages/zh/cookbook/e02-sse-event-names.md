---
title: "E02. 在 SSE 中使用命名事件"
order: 49
status: "draft"
---

SSE 允许你在同一个流上发送多种类型的事件。用 `event:` 字段给每一个起个名字，客户端就可以按类型分派到不同的处理函数。非常适合像聊天应用中的“新消息”、“用户加入”、“用户离开”这类场景。

## 发送带名称的事件

```cpp
auto send_event = [](httplib::DataSink &sink,
                     const std::string &event,
                     const std::string &data) {
  std::string msg = "event: " + event + "\n"
                  + "data: " + data + "\n\n";
  sink.write(msg.data(), msg.size());
};

svr.Get("/chat/stream", [&](const httplib::Request &req, httplib::Response &res) {
  res.set_chunked_content_provider(
    "text/event-stream",
    [&, send_event](size_t offset, httplib::DataSink &sink) {
      send_event(sink, "message", "Hello!");
      std::this_thread::sleep_for(std::chrono::seconds(2));
      send_event(sink, "join", "alice");
      std::this_thread::sleep_for(std::chrono::seconds(2));
      send_event(sink, "leave", "bob");
      std::this_thread::sleep_for(std::chrono::seconds(2));
      return true;
    });
});
```

一条消息是 `event:` → `data:` → 空行。如果你省略 `event:`，客户端会把它当作默认的 `"message"` 事件。

## 为重连附加 ID

当你包含一个 `id:` 字段时，客户端会在重连时自动把它作为 `Last-Event-ID` 发回，告诉服务端“我进行到这里”。

```cpp
auto send_event = [](httplib::DataSink &sink,
                     const std::string &event,
                     const std::string &data,
                     const std::string &id) {
  std::string msg = "id: " + id + "\n"
                  + "event: " + event + "\n"
                  + "data: " + data + "\n\n";
  sink.write(msg.data(), msg.size());
};

send_event(sink, "message", "Hello!", "42");
```

ID 的格式由你决定。单调递增的计数器或 UUID 都可以——只要在服务端选一个既唯一又可排序的东西。详情参见 [E03. 处理 SSE 重连](../e03-sse-reconnect)。

## 在 data 中放 JSON 负载

对于结构化数据，常规做法是把 JSON 放进 `data:`。

```cpp
nlohmann::json payload = {
  {"user", "alice"},
  {"text", "Hello!"},
};
send_event(sink, "message", payload.dump(), "42");
```

在客户端，将到达的 `data` 解析为 JSON，以取回原始对象。

## 带换行的数据

如果数据值包含换行，把它拆到多个 `data:` 行上。

```cpp
std::string msg = "data: line1\n"
                  "data: line2\n"
                  "data: line3\n\n";
sink.write(msg.data(), msg.size());
```

在客户端侧，这些会作为一个带有换行的单一 `data` 字符串返回。

> **提示：** 使用 `event:` 能让客户端分派更清晰，但它也有助于浏览器 DevTools——事件按类型过滤更容易。调试时这一点比你想象的更重要。
