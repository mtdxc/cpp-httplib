---
title: "S07. 以流的方式接收 multipart 数据"
order: 26
status: "draft"
---

一个简陋的上传处理函数会把整个请求塞进 `req.body`，对于大文件会撑爆内存。使用 `HandlerWithContentReader` 来逐块接收请求体。

## 基本用法

```cpp
svr.Post("/upload",
  [](const httplib::Request &req, httplib::Response &res,
     const httplib::ContentReader &content_reader) {
    if (req.is_multipart_form_data()) {
      content_reader(
        // headers of each part
        [&](const httplib::FormData &file) {
          std::cout << "name: " << file.name
                    << ", filename: " << file.filename << std::endl;
          return true;
        },
        // body of each part (called multiple times)
        [&](const char *data, size_t len) {
          // write to disk here, for example
          return true;
        });
    } else {
      // plain request body
      content_reader([&](const char *data, size_t len) {
        return true;
      });
    }

    res.set_content("ok", "text/plain");
  });
```

`content_reader` 有两种调用形式。对于 multipart 数据，传入两个回调（一个给请求头，一个给请求体）。对于普通请求体，只传一个。

## 直接写入磁盘

下面是如何将上传的文件流式写入磁盘。

```cpp
svr.Post("/upload",
  [](const httplib::Request &req, httplib::Response &res,
     const httplib::ContentReader &content_reader) {
    std::ofstream ofs;

    content_reader(
      [&](const httplib::FormData &file) {
        if (!file.filename.empty()) {
          ofs.open("uploads/" + file.filename, std::ios::binary);
        }
        return static_cast<bool>(ofs);
      },
      [&](const char *data, size_t len) {
        ofs.write(data, len);
        return static_cast<bool>(ofs);
      });

    res.set_content("uploaded", "text/plain");
  });
```

任一时刻只有一小块数据常驻内存，因此 GB 级的文件也没问题。

## 自己统计部件数

对部件（part）数量有一个上限 `CPPHTTPLIB_MULTIPART_FORM_DATA_FILE_MAX_COUNT`（默认 1024），但它只适用于缓冲路径，即每个部件都累加到 `req.form` 的情况。`ContentReader` 在库这一侧什么都不保留，因此该上限在这里不适用。

如果你想要一个上界，自己统计部件数并从请求头回调返回 `false`。解析器会立即在那里停止。

```cpp
svr.Post("/upload",
  [](const httplib::Request &req, httplib::Response &res,
     const httplib::ContentReader &content_reader) {
    size_t count = 0;

    auto ok = content_reader(
      [&](const httplib::FormData &file) {
        if (++count > 100) { return false; } // stop here
        return true;
      },
      [&](const char *data, size_t len) {
        return true;
      });

    if (!ok) {
      res.status = httplib::StatusCode::BadRequest_400;
      return;
    }

    res.set_content("ok", "text/plain");
  });
```

当 `content_reader` 返回 `false` 时，你自己设置响应状态码。请求体的其余部分不会被读取，连接会关闭，因此仍在发送数据的客户端，会看到连接被断开。

> **警告：** 当你使用 `HandlerWithContentReader` 时，`req.body` 保持**为空**。你需要自己在回调中处理请求体。

> 关于 multipart 上传的客户端侧，参见 [C07. 以 Multipart 表单数据上传文件](../c07-multipart-upload)。
