---
title: "S05. 在响应中流式发送大文件"
order: 24
status: "draft"
---

当响应是一个巨大文件或动态生成的数据时，把整个东西加载到内存并不现实。使用 `Response::set_content_provider()` 在发送时逐块生成数据。

## 当大小已知时

```cpp
svr.Get("/download", [](const httplib::Request &req, httplib::Response &res) {
  size_t total_size = get_file_size("large.bin");

  res.set_content_provider(
    total_size, "application/octet-stream",
    [](size_t offset, size_t length, httplib::DataSink &sink) {
      auto data = read_range_from_file("large.bin", offset, length);
      sink.write(data.data(), data.size());
      return true;
    });
});
```

lambda 会带着 `offset` 和 `length` 反复被调用。只读取那个范围并写入 `sink`。任一时刻只有一小块数据常驻内存。

## 发送一个文件

如果你只想发送一个文件，`set_file_content()` 简单得多。

```cpp
svr.Get("/download", [](const httplib::Request &req, httplib::Response &res) {
  res.set_file_content("large.bin", "application/octet-stream");
});
```

它在内部进行流式传输，因此即使是巨大的文件也安全。当省略 Content-Type 时，它会根据扩展名猜测。

## 当大小未知时——分块传输

对于即时生成、无法预先知道总大小的数据，使用 `set_chunked_content_provider()`。它以 HTTP 分块传输编码发送。

```cpp
svr.Get("/events", [](const httplib::Request &req, httplib::Response &res) {
  res.set_chunked_content_provider(
    "text/plain",
    [](size_t offset, httplib::DataSink &sink) {
      auto chunk = produce_next_chunk();
      if (chunk.empty()) {
        sink.done(); // done sending
        return true;
      }
      sink.write(chunk.data(), chunk.size());
      return true;
    });
});
```

调用 `sink.done()` 表示发送结束。

> **提示：** provider lambda 会被多次调用。注意被捕获变量的生命周期——必要时用 `std::shared_ptr` 包裹它们。

> 要以下载方式提供文件，参见 [S06. 返回文件下载响应](../s06-download-response)。
