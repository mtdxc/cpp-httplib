---
title: "S02. 接收 JSON 请求并返回 JSON 响应"
order: 21
status: "draft"
---

cpp-httplib 不自带 JSON 解析器。在服务端，把它与像 [nlohmann/json](https://github.com/nlohmann/json) 这样的库组合使用。下面的示例使用 `nlohmann/json`。

## 接收并返回 JSON

```cpp
#include <httplib.h>
#include <nlohmann/json.hpp>

int main() {
  httplib::Server svr;

  svr.Post("/api/users", [](const httplib::Request &req, httplib::Response &res) {
    try {
      auto in = nlohmann::json::parse(req.body);

      nlohmann::json out = {
        {"id", 42},
        {"name", in["name"]},
        {"created_at", "2026-04-10T12:00:00Z"},
      };

      res.status = 201;
      res.set_content(out.dump(), "application/json");
    } catch (const std::exception &e) {
      res.status = 400;
      res.set_content("{\"error\":\"invalid json\"}", "application/json");
    }
  });

  svr.listen("0.0.0.0", 8080);
}
```

`req.body` 是一个普通的 `std::string`，因此你可以直接把它传给 JSON 库。对于响应，`dump()` 成字符串，并把 Content-Type 设为 `application/json`。

## 检查 Content-Type

```cpp
svr.Post("/api/users", [](const httplib::Request &req, httplib::Response &res) {
  auto content_type = req.get_header_value("Content-Type");
  if (content_type.find("application/json") == std::string::npos) {
    res.status = 415; // Unsupported Media Type
    return;
  }
  // ...
});
```

当你严格只想要 JSON 时，预先验证 Content-Type。

## 一个返回 JSON 响应的辅助函数

如果你在反复写同样的写法，一个小小的辅助函数能省去打很多字。

```cpp
auto send_json = [](httplib::Response &res, int status, const nlohmann::json &j) {
  res.status = status;
  res.set_content(j.dump(), "application/json");
};

svr.Get("/api/health", [&](const auto &req, auto &res) {
  send_json(res, 200, {{"status", "ok"}});
});
```

> **提示：** 一个大的 JSON 请求体会完整地落在 `req.body` 里，这意味着它全部常驻内存。对于巨大负载，考虑流式接收——参见 [S07. 以流的方式接收 multipart 数据](../s07-multipart-reader)。

> 关于客户端侧，参见 [C02. 发送与接收 JSON](../c02-json)。
