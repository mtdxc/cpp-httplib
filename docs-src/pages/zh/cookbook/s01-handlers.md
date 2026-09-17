---
title: "S01. 注册 GET / POST / PUT / DELETE 处理函数"
order: 20
status: "draft"
---

用 `httplib::Server`，你按 HTTP 方法注册处理函数。只需把一个模式和一个 lambda 传给 `Get()`、`Post()`、`Put()` 或 `Delete()`。使用 `CustomRoute()`来处理内置集合之外的方法，比如 WebDAV 的 `PROPFIND`。

## 基本用法

```cpp
#include <httplib.h>

int main() {
  httplib::Server svr;

  svr.Get("/hello", [](const httplib::Request &req, httplib::Response &res) {
    res.set_content("Hello, World!", "text/plain");
  });

  svr.Post("/api/items", [](const httplib::Request &req, httplib::Response &res) {
    // req.body holds the request body
    res.status = 201;
    res.set_content("Created", "text/plain");
  });

  svr.Put("/api/items/1", [](const httplib::Request &req, httplib::Response &res) {
    res.set_content("Updated", "text/plain");
  });

  svr.Delete("/api/items/1", [](const httplib::Request &req, httplib::Response &res) {
    res.status = 204;
  });

  svr.listen("0.0.0.0", 8080);
}
```

处理函数接受 `(const Request&, Response&)`。用 `res.set_content()` 设置请求体和 Content-Type，用 `res.status` 设置状态码。`listen()` 启动服务端并阻塞。

## 读取查询参数

```cpp
svr.Get("/search", [](const httplib::Request &req, httplib::Response &res) {
  auto q = req.get_param_value("q");
  auto limit = req.get_param_value("limit");
  res.set_content("q=" + q + ", limit=" + limit, "text/plain");
});
```

`req.get_param_value()` 从查询串中取出一个值。如果你想先检查是否存在，用 `req.has_param("q")`。

## 读取请求头

```cpp
svr.Get("/me", [](const httplib::Request &req, httplib::Response &res) {
  auto ua = req.get_header_value("User-Agent");
  res.set_content("UA: " + ua, "text/plain");
});
```

要添加一个响应头，使用 `res.set_header("Name", "Value")`。

> **提示：** `listen()` 是一个阻塞调用。要在不同的线程上运行它，用 `std::thread` 包裹它。如果你需要非阻塞启动，参见 [S18. 用 `listen_after_bind` 控制启动顺序](../s18-listen-after-bind)。

> 要使用像 `/users/:id` 这样的路径参数，参见 [S03. 使用路径参数](../s03-path-params)。

> 对于内置集合之外的方法，比如 WebDAV 的 `PROPFIND`，参见 [S23. 处理自定义 HTTP 方法](../s23-custom-methods)。
