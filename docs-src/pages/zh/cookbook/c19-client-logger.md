---
title: "C19. 在客户端设置日志器"
order: 19
status: "draft"
---

要记录客户端发出的请求和接收的响应，使用 `set_logger()`。如果你只关心错误，还有一个单独的 `set_error_logger()`。

## 记录请求与响应

```cpp
httplib::Client cli("https://api.example.com");

cli.set_logger([](const httplib::Request &req, const httplib::Response &res) {
  std::cout << req.method << " " << req.path
            << " -> " << res.status << std::endl;
});

auto res = cli.Get("/users");
```

你传给 `set_logger()` 的回调会在每个完成的请求上触发一次。你同时得到请求和响应作为参数——因此你可以记录方法、路径、状态码、请求头、请求体，或任何你需要的其他内容。

## 只捕获错误

当发生网络层错误时（像 `Error::Connection`），`set_logger()` **不会**被调用——因为没有响应可记录。对于那些情况，使用 `set_error_logger()`。

```cpp
cli.set_error_logger([](const httplib::Error &err, const httplib::Request *req) {
  std::cerr << "error: " << httplib::to_string(err);
  if (req) {
    std::cerr << " (" << req->method << " " << req->path << ")";
  }
  std::cerr << std::endl;
});
```

第二个参数 `req` 可能为空——当故障发生在请求构建之前就会出现这种情况。解引用前务必做判空。

## 两者一起使用

一个不错的模式是：成功通过一个记录，失败通过另一个记录。

```cpp
cli.set_logger([](const auto &req, const auto &res) {
  std::cout << "[ok] " << req.method << " " << req.path
            << " " << res.status << std::endl;
});

cli.set_error_logger([](const auto &err, const auto *req) {
  std::cerr << "[ng] " << httplib::to_string(err);
  if (req) std::cerr << " " << req->method << " " << req->path;
  std::cerr << std::endl;
});
```

> **提示：** 日志回调在与请求相同的线程上同步运行。其中的重活会拖慢请求——如果你需要执行耗时的操作，把它投递到一个后台队列。
