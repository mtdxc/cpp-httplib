---
title: "C07. 以 Multipart 表单数据上传文件"
order: 7
status: "draft"
---

当你想像 HTML 的 `<input type="file">` 那样发送文件时，使用 multipart 表单数据（`multipart/form-data`）。cpp-httplib 提供两套 API——`UploadFormDataItems` 和 `FormDataProviderItems`——你根据**文件大小**在两者间选择。

## 发送小文件

先把文件读到内存，再发送。对于小文件，这是最简单的路径。

```cpp
httplib::Client cli("http://localhost:8080");

std::ifstream ifs("avatar.png", std::ios::binary);
std::string content((std::istreambuf_iterator<char>(ifs)),
                     std::istreambuf_iterator<char>());

httplib::UploadFormDataItems items = {
  {"name", "Alice", "", ""},
  {"avatar", content, "avatar.png", "image/png"},
};

auto res = cli.Post("/upload", items);
```

每个 `UploadFormData` 条目都是 `{name, content, filename, content_type}`。对于纯文本字段，将 `filename` 和 `content_type` 留空。

## 流式发送大文件

为避免把整个文件加载到内存，使用 `make_file_provider()`。它会在发送时逐块读取文件——因此即使是巨大的文件也不会撑爆你的内存占用。

```cpp
httplib::Client cli("http://localhost:8080");

httplib::UploadFormDataItems items = {
  {"name", "Alice", "", ""},
};

httplib::FormDataProviderItems provider_items = {
  httplib::make_file_provider("video", "large-video.mp4", "", "video/mp4"),
};

auto res = cli.Post("/upload", httplib::Headers{}, items, provider_items);
```

`make_file_provider()` 的参数为 `(表单字段名, 文件路径, 文件名, content type)`。将文件名留空即可原样使用文件路径。

> **提示：** 你可以在同一个请求中混合使用 `UploadFormDataItems` 和 `FormDataProviderItems`。一个清晰的分法是：文本字段放在 `UploadFormDataItems`，文件放在 `FormDataProviderItems`。

> 要展示上传进度，参见 [C11. 使用进度回调](../c11-progress-callback)。
