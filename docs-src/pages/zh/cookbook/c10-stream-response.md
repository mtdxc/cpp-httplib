---
title: "C10. 以流的方式接收响应"
order: 10
status: "draft"
---

要逐块接收响应体，使用 `ContentReceiver`。对于大文件这是显而易见的首选，但对于 NDJSON（换行分隔的 JSON）或日志流——你想在数据到达时就开始处理——它同样好用。

## 处理每一块数据

```cpp
httplib::Client cli("http://localhost:8080");

auto res = cli.Get("/logs/stream",
  [](const char *data, size_t len) {
    std::cout.write(data, len);
    std::cout.flush();
    return true; // return false to stop receiving
  });
```

数据按从服务端接收的顺序在 lambda 中到达。从回调返回 `false` 可以在下载中途停止。

## 逐行解析 NDJSON

下面是一种缓冲方式，用于逐行处理换行分隔的 JSON。

```cpp
std::string buffer;

auto res = cli.Get("/events",
  [&](const char *data, size_t len) {
    buffer.append(data, len);
    size_t pos;
    while ((pos = buffer.find('\n')) != std::string::npos) {
      auto line = buffer.substr(0, pos);
      buffer.erase(0, pos + 1);
      if (!line.empty()) {
        auto j = nlohmann::json::parse(line);
        handle_event(j);
      }
    }
    return true;
  });
```

累加到一个缓冲区，然后每次看到换行符就取出一行并解析。这是实时消费流式 API 的标准写法。

> **警告：** 当你传入一个 `ContentReceiver` 时，`res->body` 会保持**为空**。你需要自己在回调中存储或处理请求体。

> 要跟踪下载进度，将此与 [C11. 使用进度回调](../c11-progress-callback) 结合使用。
> 关于 Server-Sent Events（SSE），参见 [E04. 在客户端接收 SSE](../e04-sse-client)。
