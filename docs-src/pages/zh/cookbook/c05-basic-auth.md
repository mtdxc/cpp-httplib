---
title: "C05. 使用 Basic 认证"
order: 5
status: "draft"
---

对于需要 Basic 认证的端点，将用户名和密码传给 `set_basic_auth()`。cpp-httplib 会为你构建 `Authorization: Basic ...` 请求头。

## 基本用法

```cpp
httplib::Client cli("https://api.example.com");
cli.set_basic_auth("alice", "s3cret");

auto res = cli.Get("/private");
if (res && res->status == 200) {
  std::cout << res->body << std::endl;
}
```

设置一次，该客户端的每个请求都会携带凭据。无需每次都重新构建请求头。

## 单请求级用法

如果你只想在某个特定请求上带上凭据，直接传入请求头即可。

```cpp
httplib::Headers headers = {
  httplib::make_basic_authentication_header("alice", "s3cret"),
};
auto res = cli.Get("/private", headers);
```

`make_basic_authentication_header()` 会为你构建 Base64 编码的请求头。

> **警告：** Basic 认证只是对凭据进行 Base64 **编码**——并不会加密它们。务必在 HTTPS 下使用。在普通 HTTP 上，你的密码会在网络上明文传输。

## Digest 认证

对于更安全的 Digest 认证机制，使用 `set_digest_auth()`。仅当 cpp-httplib 使用 OpenSSL（或其他 TLS 后端）构建时，该功能才可用。

```cpp
cli.set_digest_auth("alice", "s3cret");
```

> 要使用 Bearer token 调用 API，参见 [C06. 使用 Bearer token 调用 API](../c06-bearer-token)。
