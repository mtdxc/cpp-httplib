---
title: "C02. 发送与接收 JSON"
order: 2
status: "draft"
---

cpp-httplib 不自带 JSON 解析器。请使用像 [nlohmann/json](https://github.com/nlohmann/json) 这样的库来构建和解析 JSON。这里的示例使用 `nlohmann/json`。

## 发送 JSON

```cpp
httplib::Client cli("http://localhost:8080");

nlohmann::json j = {{"name", "Alice"}, {"age", 30}};
auto res = cli.Post("/api/users", j.dump(), "application/json");
```

将 JSON 字符串作为第二个参数传给 `Post()`，将 Content-Type 作为第三个参数。同样的写法也适用于 `Put()` 和 `Patch()`。

> **警告：** 如果省略 Content-Type（第三个参数），服务端可能无法将请求体识别为 JSON。请务必指定 `"application/json"`。

## 接收 JSON 响应

```cpp
auto res = cli.Get("/api/users/1");
if (res && res->status == 200) {
  auto j = nlohmann::json::parse(res->body);
  std::cout << j["name"] << std::endl;
}
```

`res->body` 是一个 `std::string`，你可以直接把它传给 JSON 库。

> **提示：** 服务端有时会在出错时返回 HTML。为稳妥起见，解析前先检查状态码。某些 API 还要求带上 `Accept: application/json` 请求头。如果你要反复调用某个 JSON API，[C03. 设置默认请求头](../c03-default-headers) 可以省去一些样板代码。

> 关于如何在服务端接收并返回 JSON，参见 [S02. 接收 JSON 请求和返回 JSON 响应](../s02-json-api)。
