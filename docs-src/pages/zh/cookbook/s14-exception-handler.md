---
title: "S14. 捕获异常"
order: 33
status: "draft"
---

当一个路由处理函数抛出异常时，cpp-httplib 会让服务端继续运行并以 500 响应。不过默认情况下，很少有错误信息会传递到客户端。`set_exception_handler()` 让你拦截异常并构建你自己的响应。

## 基本用法

```cpp
svr.set_exception_handler(
  [](const httplib::Request &req, httplib::Response &res,
     std::exception_ptr ep) {
    try {
      std::rethrow_exception(ep);
    } catch (const std::exception &e) {
      res.status = 500;
      res.set_content(std::string("error: ") + e.what(), "text/plain");
    } catch (...) {
      res.status = 500;
      res.set_content("unknown error", "text/plain");
    }
  });
```

处理函数接收一个 `std::exception_ptr`。惯用的做法是用 `std::rethrow_exception()` 重新抛出并按类型捕获。你可以根据异常类型变化状态码和消息。

## 按自定义异常类型分支

如果你抛出自己的异常类型，你可以把它们映射到 400 或 404 响应。

```cpp
struct NotFound : std::runtime_error {
  using std::runtime_error::runtime_error;
};
struct BadRequest : std::runtime_error {
  using std::runtime_error::runtime_error;
};

svr.set_exception_handler(
  [](const auto &req, auto &res, std::exception_ptr ep) {
    try {
      std::rethrow_exception(ep);
    } catch (const NotFound &e) {
      res.status = 404;
      res.set_content(e.what(), "text/plain");
    } catch (const BadRequest &e) {
      res.status = 400;
      res.set_content(e.what(), "text/plain");
    } catch (const std::exception &e) {
      res.status = 500;
      res.set_content("internal error", "text/plain");
    }
  });
```

现在，在处理函数内抛 `NotFound("user not found")` 就足以返回 404。无需每个处理函数都写 try/catch。

## 与 set_error_handler 的关系

`set_exception_handler()` 在异常抛出的那一刻运行。之后，如果 `res.status` 是 4xx 或 5xx，`set_error_handler()` 也会运行。顺序是 `exception_handler` → `error_handler`。把它们的角色理解为：

- **异常处理函数**：解读异常，设置状态码和消息
- **错误处理函数**：看到状态码，把它包裹进共享的模板

> **提示：** 没有异常处理函数时，cpp-httplib 返回一个默认的 500 响应，异常详情永远进不了日志。对你想要调试的任何东西，都务必设置一个。
