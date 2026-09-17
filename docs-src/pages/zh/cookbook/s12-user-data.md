---
title: "S12. 用 res.user_data 在处理函数间传递数据"
order: 31
status: "draft"
---

假设你的 pre-request 处理函数解码了一个认证 token，你希望路由处理函数使用这个结果。这种“处理函数之间的数据交接”就是 `res.user_data` 的用途——它能保存任意类型的值。

## 基本用法

```cpp
struct AuthUser {
  std::string id;
  std::string name;
  bool is_admin;
};

svr.set_pre_request_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    auto token = req.get_header_value("Authorization");
    auto user = decode_token(token); // decode the auth token
    res.user_data.set("user", user);
    return httplib::Server::HandlerResponse::Unhandled;
  });

svr.Get("/me", [](const httplib::Request &req, httplib::Response &res) {
  auto *user = res.user_data.get<AuthUser>("user");
  if (!user) {
    res.status = 401;
    return;
  }
  res.set_content("Hello, " + user->name, "text/plain");
});
```

`user_data.set()` 存储一个任意类型的值，`user_data.get<T>()` 取回它。如果你给错了类型，会拿回 `nullptr`——所以要小心。

## 典型的值类型

字符串、数字、结构体、`std::shared_ptr`——任何可拷贝或可移动的东西都行。

```cpp
res.user_data.set("user_id", std::string{"42"});
res.user_data.set("is_admin", true);
res.user_data.set("started_at", std::chrono::steady_clock::now());
```

## 在哪里设置，在哪里读取

常见的流程是：在 `set_pre_routing_handler()` 或 `set_pre_request_handler()` 中设置，在路由处理函数中读取。Pre-request 在路由之后运行，所以你可以把它与 `req.matched_route` 结合，只为特定路由设置值。

## 一个坑

`user_data` 活在 `Response` 上，而不是 `Request` 上。这是因为处理函数拿到的是 `Response&`（可变的），但只有 `const Request&`。乍看之下很奇怪，但一旦你把它理解为“处理函数之间共享的可变上下文”就说得通了。

> **警告：** 当类型不匹配时，`user_data.get<T>()` 返回 `nullptr`。在 set 和 get 上使用完全相同的类型。以 `AuthUser` 存储、以 `const AuthUser` 取回是行不通的。
