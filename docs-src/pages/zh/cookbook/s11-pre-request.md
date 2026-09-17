---
title: "S11. 用 pre-request 处理函数逐路由认证"
order: 30
status: "draft"
---

[S09. 为所有路由添加前置处理](../s09-pre-routing) 中的 `set_pre_routing_handler()` 在**路由之前**运行，所以它不知道哪个路由匹配了。当你想要逐路由的行为时，`set_pre_request_handler()` 才是你需要的。

## Pre-routing vs. pre-request

| 钩子 | 何时运行 | 路由信息 | 请求体 |
| --- | --- | --- | --- |
| `set_pre_routing_handler` | 路由之前 | 不可用 | 尚未读取 |
| `set_pre_request_handler` | 路由之后、紧接在路由处理函数之前 | 通过 `req.matched_route` 可用 | 尚未读取 |

在 pre-request 处理函数中，`req.matched_route` 保存匹配到的**模式字符串**。你可以根据路由定义本身来变化行为。

由于 pre-request 处理函数运行时请求体尚未读取，你可以在不消费（可能很大的）请求体的情况下拒绝一个请求——比如在认证检查失败时。注意这也意味着 `req.body` 和从请求体解析出的表单字段在这里不可用；转而检查请求头、路径、查询参数或 `req.matched_route`。

## 按路由切换认证

```cpp
svr.set_pre_request_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    // require auth for routes starting with /admin
    if (req.matched_route.rfind("/admin", 0) == 0) {
      auto token = req.get_header_value("Authorization");
      if (!is_admin_token(token)) {
        res.status = 403;
        res.set_content("forbidden", "text/plain");
        return httplib::Server::HandlerResponse::Handled;
      }
    }
    return httplib::Server::HandlerResponse::Unhandled;
  });
```

`matched_route` 是在路径参数展开**之前**的模式（例如 `/admin/users/:id`）。你是拿路由定义来比较，而不是拿实际的请求路径，因此 ID 或名称不会干扰你。

## 返回值

与 pre-routing 相同——返回 `HandlerResponse`。

- `Unhandled`：继续（路由处理函数会运行）
- `Handled`：到此为止，跳过路由处理函数

## 把认证信息传给路由处理函数

要把解码后的用户信息传进路由处理函数，使用 `res.user_data`。参见 [S12. 用 `res.user_data` 在处理函数间传递数据](../s12-user-data)。
