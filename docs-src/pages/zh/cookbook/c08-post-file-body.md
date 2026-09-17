---
title: "C08. 以原始二进制 POST 文件"
order: 8
status: "draft"
---

有时你想把文件内容直接作为请求体发送——不用 multipart 包裹。这对于 S3 兼容的 API 或接受原始图像数据的端点很常见。为此，使用 `make_file_body()`。

## 基本用法

```cpp
httplib::Client cli("https://storage.example.com");

auto [size, provider] = httplib::make_file_body("backup.tar.gz");
if (size == 0) {
  std::cerr << "Failed to open file" << std::endl;
  return 1;
}

auto res = cli.Put("/bucket/backup.tar.gz", size,
                   provider, "application/gzip");
```

`make_file_body()` 返回一个由文件大小和 `ContentProvider` 组成的 pair。把它们传给 `Post()` 或 `Put()`，文件内容就会直接灌入请求体。

`ContentProvider` 会逐块读取文件，因此即使是巨大的文件也绝不会完整地常驻内存。

## 当文件无法打开时

如果文件无法打开，`make_file_body()` 会把 `size` 返回为 `0`，`provider` 为一个空的函数对象。发送它会产生垃圾数据——务必先检查 `size`。

> **警告：** `make_file_body()` 需要预先确定 Content-Length，因此它会提前读取文件大小。如果文件大小可能在上传过程中变化，这个 API 并不合适。

> 要改为以 multipart 表单数据发送文件，参见 [C07. 以 Multipart 表单数据上传文件](../c07-multipart-upload)。
