---
title: "S10. 用 post-routing 处理函数添加响应头"
order: 29
status: "draft"
---

有时你希望在处理函数运行后向响应添加共享的请求头——CORS 头、安全头、一个请求 ID 等等。这就是 `set_post_routing_handler()` 的用途。

## 基本用法

```cpp
svr.set_post_routing_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    res.set_header("X-Request-ID", generate_request_id());
  });
```

post-routing 处理函数在**路由处理函数之后、响应发送之前**运行。在这里你可以调用 `res.set_header()` 或 `res.headers.erase()`，在一处为每个响应添加或移除请求头。

## 添加 CORS 头

CORS 是一个典型用例。

```cpp
svr.set_post_routing_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    res.set_header("Access-Control-Allow-Origin", "*");
    res.set_header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
    res.set_header("Access-Control-Allow-Headers", "Content-Type, Authorization");
  });
```

对于预检 `OPTIONS` 请求，注册一个单独的处理函数——或者在 pre-routing 处理函数中处理它们。

```cpp
svr.Options("/.*", [](const auto &req, auto &res) {
  res.status = 204;
});
```

## 集中管理你的安全头

在一处管理浏览器安全头。

```cpp
svr.set_post_routing_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    res.set_header("X-Content-Type-Options", "nosniff");
    res.set_header("X-Frame-Options", "DENY");
    res.set_header("Referrer-Policy", "strict-origin-when-cross-origin");
  });
```

无论哪个处理函数产生了响应，同样的请求头都会被附上。

> **提示：** post-routing 处理函数也会对那些未匹配任何路由的响应以及来自错误处理函数的响应运行。当你需要确保每个响应都带上某些请求头时，这正是你想要的。
