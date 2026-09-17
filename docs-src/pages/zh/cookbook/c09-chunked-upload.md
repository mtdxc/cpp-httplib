---
title: "C09. 以分块传输发送请求体"
order: 9
status: "draft"
---

当你无法预先知道请求体大小——比如即时生成的数据或从另一个流管道过来的数据——使用 `ContentProviderWithoutLength`。客户端会用 HTTP 分块传输编码发送请求体。

## 基本用法

```cpp
httplib::Client cli("http://localhost:8080");

auto res = cli.Post("/stream",
  [&](size_t offset, httplib::DataSink &sink) {
    std::string chunk = produce_next_chunk();
    if (chunk.empty()) {
      sink.done(); // done sending
      return true;
    }
    return sink.write(chunk.data(), chunk.size());
  },
  "application/octet-stream");
```

lambda 的任务就是：生成下一块数据并用 `sink.write()` 发送。当没有更多数据时，调用 `sink.done()` 就完成了。

## 当大小已知时

如果你**确实**能提前知道总大小，使用 `ContentProvider` 重载（接受 `size_t offset, size_t length, DataSink &sink`）并同时传入总大小。

```cpp
size_t total_size = get_total_size();

auto res = cli.Post("/upload", total_size,
  [&](size_t offset, size_t length, httplib::DataSink &sink) {
    auto data = read_range(offset, length);
    return sink.write(data.data(), data.size());
  },
  "application/octet-stream");
```

在大小已知的情况下，请求会携带 Content-Length 请求头——因此服务端可以展示进度。能做到的话优先用这种形式。

> **细节：** `sink.write()` 返回一个 `bool`，表示写入是否成功。如果它返回 `false`，说明连接已断开——从 lambda 返回 `false` 以停止。

> 如果你只是发送一个文件，`make_file_body()` 更简单。参见 [C08. 以原始二进制 POST 文件](../c08-post-file-body)。
