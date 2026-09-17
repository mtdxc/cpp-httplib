---
title: "S09. 为所有路由添加前置处理"
order: 28
status: "draft"
---

有时你希望同样的逻辑在每个请求前运行——认证检查、日志、限流。用 `set_pre_routing_handler()` 注册这些。

## 基本用法

```cpp
svr.set_pre_routing_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    std::cout << req.method << " " << req.path << std::endl;
    return httplib::Server::HandlerResponse::Unhandled;
  });
```

pre-routing 处理函数在**路由之前**运行。它会捕获每个请求——包括那些不匹配任何处理函数的请求。

`HandlerResponse` 返回值是关键：

- 返回 `Unhandled` → 继续正常流程（路由和实际的处理函数都会运行）
- 返回 `Handled` → 响应被视为已完成，跳过后续

## 用它做认证

把你共享的认证检查集中在一处。

```cpp
svr.set_pre_routing_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    if (req.path.rfind("/public", 0) == 0) {
      return httplib::Server::HandlerResponse::Unhandled; // no auth needed
    }

    auto auth = req.get_header_value("Authorization");
    if (auth.empty()) {
      res.status = 401;
      res.set_content("unauthorized", "text/plain");
      return httplib::Server::HandlerResponse::Handled;
    }

    return httplib::Server::HandlerResponse::Unhandled;
  });
```

如果认证失败，返回 `Handled` 以立即用 401 响应。如果通过，返回 `Unhandled` 并把控制权交给路由。

## 用于逐路由认证

如果你想要按路由设置不同的认证规则而不是单一的全局检查，`set_pre_request_handler()` 更合适。参见 [S11. 用 pre-request 处理函数逐路由认证](../s11-pre-request)。

> **提示：** 如果你只想修改响应，`set_post_routing_handler()` 才是合适的工具。参见 [S10. 用 post-routing 处理函数添加响应头](../s10-post-routing)。
