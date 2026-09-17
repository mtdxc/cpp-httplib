---
title: "C01. 获取响应体 / 保存到文件"
order: 1
status: "draft"
---

## 以字符串获取

```cpp
httplib::Client cli("http://localhost:8080");
auto res = cli.Get("/hello");
if (res && res->status == 200) {
  std::cout << res->body << std::endl;
}
```

`res->body` 是一个 `std::string`，可直接使用。整个响应会被加到内存中。

> **警告：** 如果用 `res->body` 拉取大文件，它们全部会进入内存。大文件下载请使用下面的 `ContentReceiver`。

## 保存到文件

```cpp
httplib::Client cli("http://localhost:8080");

std::ofstream ofs("output.bin", std::ios::binary);
if (!ofs) {
  std::cerr << "Failed to open file" << std::endl;
  return 1;
}

auto res = cli.Get("/large-file",
  [&](const char *data, size_t len) {
    ofs.write(data, len);
    return static_cast<bool>(ofs);
  });
```

使用 `ContentReceiver` 时，数据分块到达。你可以把每块直接写盘，无需在内存中缓存整个响应体——非常适合大文件下载。

回调返回 `false` 会中止下载。上面的例子中，如果 `ofs` 写入失败，下载会自动停止。

> **细节：** 想在下载前检查 Content-Length 这样的响应头？将 `ResponseHandler` 与 `ContentReceiver` 结合起来用。
>
> ```cpp
> auto res = cli.Get("/large-file",
>   [](const httplib::Response &res) {
>     auto len = res.get_header_value("Content-Length");
>     std::cout << "Size: " << len << std::endl;
>     return true; // return false to skip the download
>   },
>   [&](const char *data, size_t len) {
>     ofs.write(data, len);
>     return static_cast<bool>(ofs);
>   });
> ```
>
> `ResponseHandler` 在head到达后、body返回前调用。返回 `false` 可完全跳过下载。

> 要显示下载进度，参见 [C11. 使用进度回调](../c11-progress-callback)。
