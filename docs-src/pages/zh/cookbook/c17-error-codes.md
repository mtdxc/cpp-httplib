---
title: "C17. 处理错误码"
order: 17
status: "draft"
---

`cli.Get()`、`cli.Post()` 及同类函数返回一个 `Result`。当请求失败时——无法连接服务端、超时等——结果是“假值”（falsy）。要获取具体原因，使用 `Result::error()`。

## 基本检查

```cpp
httplib::Client cli("http://localhost:8080");
auto res = cli.Get("/api/data");

if (res) {
  // the request was sent and a response came back
  std::cout << "status: " << res->status << std::endl;
} else {
  // the network layer failed
  std::cerr << "error: " << httplib::to_string(res.error()) << std::endl;
}
```

用 `if (res)` 来检查是否成功。失败时，`res.error()` 返回一个 `httplib::Error` 枚举值。把它传给 `to_string()` 以获得可读的描述。

## 常见错误

| 值 | 含义 |
| --- | --- |
| `Error::Connection` | 无法连接到服务端 |
| `Error::ConnectionTimeout` | 连接超时（`set_connection_timeout`） |
| `Error::Read` / `Error::Write` | 发送或接收期间出错 |
| `Error::Timeout` | 通过 `set_max_timeout` 设置的整体超时 |
| `Error::ExceedRedirectCount` | 重定向次数过多 |
| `Error::SSLConnection` | TLS 握手失败 |
| `Error::SSLServerVerification` | 服务端证书验证失败 |
| `Error::Canceled` | 一个进度回调返回了 `false` |

## 网络错误 vs. HTTP 错误

即使 `res` 为真值，HTTP 状态码仍可能是 4xx 或 5xx。这是两回事。

```cpp
auto res = cli.Get("/api/data");
if (!res) {
  // network error (no response received at all)
  std::cerr << "network error: " << httplib::to_string(res.error()) << std::endl;
  return 1;
}

if (res->status >= 400) {
  // HTTP error (response received, but the status is bad)
  std::cerr << "http error: " << res->status << std::endl;
  return 1;
}

// success
std::cout << res->body << std::endl;
```

在心里把它们分开：网络层错误走 `res.error()`，HTTP 层错误走 `res->status`。

> 要深入了解与 SSL 相关的错误，参见 [C18. 处理 SSL 错误](../c18-ssl-errors)。
