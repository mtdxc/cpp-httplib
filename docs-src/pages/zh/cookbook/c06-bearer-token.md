---
title: "C06. 使用 Bearer Token 调用 API"
order: 6
status: "draft"
---

对于 Bearer token 认证——在 OAuth 2.0 和现代 Web API 中很常见——使用 `set_bearer_token_auth()`。传入 token，cpp-httplib 会为你构建 `Authorization: Bearer <token>` 请求头。

## 基本用法

```cpp
httplib::Client cli("https://api.example.com");
cli.set_bearer_token_auth("eyJhbGciOiJIUzI1NiIs...");

auto res = cli.Get("/me");
if (res && res->status == 200) {
  std::cout << res->body << std::endl;
}
```

设置一次，之后的每个请求都会携带 token。对于像 GitHub、Slack 或你自己的 OAuth 服务这类基于 token 的 API，这是常用的写法。

## 单请求级用法

当你只想在某个请求上带 token——或者需要每请求一个不同的 token——通过请求头传入。

```cpp
httplib::Headers headers = {
  httplib::make_bearer_token_authentication_header(token),
};
auto res = cli.Get("/me", headers);
```

`make_bearer_token_authentication_header()` 会为你构建 `Authorization` 请求头。

## 刷新 token

当 token 过期时，只需用新 token 再次调用 `set_bearer_token_auth()` 即可。

```cpp
if (res && res->status == 401) {
  auto new_token = refresh_token();
  cli.set_bearer_token_auth(new_token);
  res = cli.Get("/me");
}
```

> **警告：** Bearer token 本身就是一种凭据。务必通过 HTTPS 发送它，并且绝不要把它硬编码到源码或配置文件中。

> 要一次设置多个请求头，参见 [C03. 设置默认请求头](../c03-default-headers)。
