---
title: "C04. 跟随重定向"
order: 4
status: "draft"
---

默认情况下，cpp-httplib 不会进行 HTTP 重定向（3xx）。如果服务端返回 `302 Found`，你得到的只是一个状态码为 302 的响应——仅此而已。

要自动跟随重定向，调用 `set_follow_location(true)`。

## 跟随重定向

```cpp
httplib::Client cli("http://example.com");
cli.set_follow_location(true);

auto res = cli.Get("/old-path");
if (res && res->status == 200) {
  std::cout << res->body << std::endl;
}
```

启用 `set_follow_location(true)` 后，客户端会读取 `Location` 请求头，并自动向新 URL 重新发出请求。最终响应会落在 `res` 中。

## 从 HTTP 重定向到 HTTPS

```cpp
httplib::Client cli("http://example.com");
cli.set_follow_location(true);

auto res = cli.Get("/");
```

许多站点会把 HTTP 流量重定向到 HTTPS。开启 `set_follow_location(true)` 后，这种情况会被透明处理——即使协议或主机发生变化，客户端也会跟着重定向。

> **警告：** 要重定向到 HTTPS，你需要使用 OpenSSL（或其他 TLS 后端）来构建 cpp-httplib。若没有 TLS 支持，重定向到 HTTPS 会失败。

> **提示：** 重定向会增加总的请求耗时。超时配置参见 [C12. 设置超时](../c12-timeouts)。
