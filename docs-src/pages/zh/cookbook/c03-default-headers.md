---
title: "C03. 设置默认请求头"
order: 3
status: "draft"
---

当你希望在每个请求上都带上相同的请求头时，使用 `set_default_headers()`。一旦设置，它们会自动附加到该 Client 发出的每一个请求上。

## 基本用法

```cpp
httplib::Client cli("https://api.example.com");

cli.set_default_headers({
  {"Accept", "application/json"},
  {"User-Agent", "my-app/1.0"},
});

auto res = cli.Get("/users");
```

将你在每次 API 调用中都需要的请求头（比如 `Accept` 或 `User-Agent`）在此处集中注册。无需在每个请求上重复调用。

## 在每个请求上发送 Bearer token

```cpp
httplib::Client cli("https://api.example.com");

cli.set_default_headers({
  {"Authorization", "Bearer " + token},
  {"Accept", "application/json"},
});

auto res1 = cli.Get("/me");
auto res2 = cli.Get("/projects");
```

设置一次认证 token，之后的每个请求都会携带它。当你编写一个需要调用多个 API 地址的客户端时，这会很方便。

> **提示：** `set_default_headers()` 会**替换**现有的默认请求头。即使你只想加一个，也要重新传入完整的集合。

## 与单请求级请求头配合使用

即使设置了默认请求头，你仍可以在单个请求上传入额外的请求头。

```cpp
httplib::Headers headers = {
  {"X-Request-ID", "abc-123"},
};
auto res = cli.Get("/users", headers);
```

单请求级请求头会**追加**在默认请求头之上。两者都会被发送到服务端。

> 关于 Bearer token 认证的详情，参见 [C06. 使用 Bearer token 调用 API](../c06-bearer-token)。
