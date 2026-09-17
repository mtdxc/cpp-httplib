---
title: "C11. 使用进度回调"
order: 11
status: "draft"
---

要展示下载或上传进度，传入一个 `DownloadProgress` 或 `UploadProgress` 回调。两者都接受两个参数：`(current, total)`。

## 下载进度

```cpp
httplib::Client cli("http://localhost:8080");

auto res = cli.Get("/large-file",
  [](size_t current, size_t total) {
    auto percent = (total > 0) ? (current * 100 / total) : 0;
    std::cout << "\rDownloading: " << percent << "% ("
              << current << "/" << total << ")" << std::flush;
    return true; // return false to abort
  });
std::cout << std::endl;
```

回调会在每次数据到达时触发。`total` 来自 Content-Length 请求头——如果服务端没有发送该头，它可能为 `0`。这种情况下无法计算百分比，那就只展示已接收的字节数。

## 上传进度

上传的做法一样。将 `UploadProgress` 作为最后一个参数传给 `Post()` 或 `Put()`。

```cpp
httplib::Client cli("http://localhost:8080");

std::string body = load_large_body();

auto res = cli.Post("/upload", body, "application/octet-stream",
  [](size_t current, size_t total) {
    auto percent = current * 100 / total;
    std::cout << "\rUploading: " << percent << "%" << std::flush;
    return true;
  });
std::cout << std::endl;
```

## 传输中途取消

从回调返回 `false` 会中止传输。这就是你在 UI 中接入“取消”按钮的方式——翻转一个标志位，下一次进度回调就会停止传输。

```cpp
std::atomic<bool> cancelled{false};

auto res = cli.Get("/large-file",
  [&](size_t current, size_t total) {
    return !cancelled.load();
  });
```

> **提示：** `ContentReceiver` 和进度回调可以同时使用。当你想一边流式写入文件一边展示进度时，把两者都传入。

> 关于保存到文件的具体示例，参见 [C01. 获取响应体 / 保存到文件](../c01-get-response-body)。
