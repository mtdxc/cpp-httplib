---
title: "S13. 返回自定义错误页面"
order: 32
status: "draft"
---

要自定义 4xx 或 5xx 错误的响应，使用 `set_error_handler()`。你可以用自己的 HTML 或 JSON 替换朴素的默认错误页。

## 基本用法

```cpp
svr.set_error_handler([](const httplib::Request &req, httplib::Response &res) {
  auto body = "<h1>Error " + std::to_string(res.status) + "</h1>";
  res.set_content(body, "text/html");
});
```

错误处理函数在一个错误响应发送前立即运行——任何时候 `res.status` 是 4xx 或 5xx。用 `res.set_content()` 替换请求体，每个错误响应都会用同样的模板。

## 按状态码分支

```cpp
svr.set_error_handler([](const httplib::Request &req, httplib::Response &res) {
  if (res.status == 404) {
    res.set_content("<h1>Not Found</h1><p>" + req.path + "</p>", "text/html");
  } else if (res.status >= 500) {
    res.set_content("<h1>Server Error</h1>", "text/html");
  }
});
```

检查 `res.status` 让你能为 404 展示一个自定义消息，为 5xx 错误展示一个“联系支持”链接。

## JSON 错误响应

对于一个 API 服务端，你大概希望错误以 JSON 返回。

```cpp
svr.set_error_handler([](const httplib::Request &req, httplib::Response &res) {
  nlohmann::json j = {
    {"error", true},
    {"status", res.status},
    {"path", req.path},
  };
  res.set_content(j.dump(), "application/json");
});
```

现在每个错误都以一致的 JSON 形状返回。

> **提示：** `set_error_handler()` 也会对由路由处理函数抛出异常导致的 500 响应触发。要获取异常本身，把它与 `set_exception_handler()` 结合使用。参见 [S14. 捕获异常](../s14-exception-handler)。
