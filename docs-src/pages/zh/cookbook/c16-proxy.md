---
title: "C16. 通过代理发送请求"
order: 16
status: "draft"
---

要把流量路由过企业网络或特定路径，通过 HTTP 代理发送请求。只需将代理的主机和端口传给 `set_proxy()`。

## 基本用法

```cpp
httplib::Client cli("https://api.example.com");
cli.set_proxy("proxy.internal", 8080);

auto res = cli.Get("/users");
```

请求会经过代理。对于 HTTPS，客户端使用 CONNECT 方法建立隧道——无需额外配置。

## 代理认证

如果代理本身需要认证，使用 `set_proxy_basic_auth()` 或 `set_proxy_bearer_token_auth()`。

```cpp
cli.set_proxy("proxy.internal", 8080);
cli.set_proxy_basic_auth("user", "password");
```

```cpp
cli.set_proxy_bearer_token_auth("token");
```

如果 cpp-httplib 使用 OpenSSL（或其他 TLS 后端）构建，你还可以为代理使用 Digest 认证。

```cpp
cli.set_proxy_digest_auth("user", "password");
```

## 与终端服务端认证配合使用

代理认证与向终端服务端认证是分开的（[C05. 使用 Basic 认证](../c05-basic-auth)、[C06. 使用 Bearer token 调用 API](../c06-bearer-token)）。当两者都需要时，把两者都设置好。

```cpp
cli.set_proxy("proxy.internal", 8080);
cli.set_proxy_basic_auth("proxy-user", "proxy-pass");

cli.set_bearer_token_auth("api-token"); // for the end server
```

`Proxy-Authorization` 发送到代理，`Authorization` 发送到终端服务端。

## 针对特定主机绕过代理

你通常会让内部端点跳过代理。用 `set_no_proxy()` 配置一个绕过列表。

```cpp
cli.set_proxy("proxy.internal", 8080);
cli.set_no_proxy({"internal.corp", "10.0.0.0/8", "*.dev.local"});
```

每个条目是以下之一：

- `*` — 为所有主机绕过代理
- 主机名后缀（例如 `example.com`）——匹配 `example.com` 本身及任意子域（`foo.example.com`）。前导点是允许的，但仅供参考；两种写法等价。
- 单个 IP 字面量（例如 `192.168.1.1`、`::1`）
- 一个 CIDR 块（例如 `10.0.0.0/8`、`fe80::/10`）

主机名匹配不区分大小写，并使用点边界规则，因此条目 `example.com` **不会**匹配 `evilexample.com`。IP 比较会通过 `inet_pton` 归一化，因此无法通过替代的字符串形式（例如 `127.000.000.001`）绕过 `127.0.0.1`。当一个条目匹配时，`Proxy-Authorization` 请求头也会被抑制。

格式错误的条目会被静默丢弃。不支持像 `example.com:8080` 这样带端口的条目（cpp-httplib 其他以主机为键的 API 也只以主机名为键）。

## 从环境变量读取代理配置

cpp-httplib 自身不会去读 `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`——配置 API 始终是显式的，就像 `set_ca_cert_path()` 一样。如果你想要那种行为，就在你的应用里读取这些变量，并把它们喂给 `set_proxy()` 和 `set_no_proxy()`。

```cpp
if (const char *v = std::getenv("no_proxy")) {
  std::vector<std::string> patterns;
  std::stringstream ss(v);
  for (std::string item; std::getline(ss, item, ',');) {
    if (!item.empty()) { patterns.push_back(item); }
  }
  cli.set_no_proxy(patterns);
}
```

如果你也自己读取 `HTTP_PROXY`，请只响应小写的 `http_proxy`。大写形式在 CGI/FastCGI 环境中会被 `Proxy:` 请求头污染（[CVE-2016-5385 / “httpoxy”](https://httpoxy.org/)）。`HTTPS_PROXY` 和 `NO_PROXY` 在任何大小写下都是安全的，因为它们的名字不以 `HTTP_` 开头。
