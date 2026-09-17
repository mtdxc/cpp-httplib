---
title: "T05. 在服务端访问对端证书"
order: 47
status: "draft"
---

在 mTLS 部署中，你可以从处理函数内读取客户端的证书。抽出 CN 或 SAN 来识别用户或记录请求。

## 基本用法

```cpp
svr.Get("/me", [](const httplib::Request &req, httplib::Response &res) {
  auto cert = req.peer_cert();
  if (!cert) {
    res.status = 401;
    res.set_content("no client certificate", "text/plain");
    return;
  }

  auto cn = cert.subject_cn();
  res.set_content("hello, " + cn, "text/plain");
});
```

`req.peer_cert()` 返回一个 `tls::PeerCert`。它可以转换为 `bool`，因此在使用前先检查是否存在证书。

## 可用的字段

从一个 `PeerCert`，你可以获取：

```cpp
auto cert = req.peer_cert();

std::string cn = cert.subject_cn();        // CN
std::string issuer = cert.issuer_name();   // issuer
std::string serial = cert.serial();        // serial number

time_t not_before, not_after;
cert.validity(not_before, not_after);      // validity period

auto sans = cert.sans();                   // SANs
for (const auto &san : sans) {
  std::cout << san.value << std::endl;
}
```

还有一个辅助函数来检查某个主机名是否被 SAN 列表覆盖：

```cpp
if (cert.check_hostname("alice.corp.example.com")) {
  // matches
}
```

## 基于证书的授权

你可以按 CN 或 SAN 来把守路由。

```cpp
svr.set_pre_request_handler(
  [](const httplib::Request &req, httplib::Response &res) {
    auto cert = req.peer_cert();
    if (!cert) {
      res.status = 401;
      return httplib::Server::HandlerResponse::Handled;
    }

    if (req.matched_route.rfind("/admin", 0) == 0) {
      auto cn = cert.subject_cn();
      if (!is_admin_cn(cn)) {
        res.status = 403;
        return httplib::Server::HandlerResponse::Handled;
      }
    }

    return httplib::Server::HandlerResponse::Unhandled;
  });
```

与一个 pre-request 处理函数结合，你可以把所有授权逻辑集中在一处。参见 [S11. 用 pre-request 处理函数逐路由认证](../s11-pre-request)。

## SNI（Server Name Indication，服务器名称指示）

cpp-httplib 会自动处理 SNI。如果一个服务端托管多个域，SNI 会在底层被使用——但通常处理函数不需要关心。

> **警告：** 仅当启用了 mTLS 且客户端确实出示了证书时，`req.peer_cert()` 才会返回有意义的值。对于普通 TLS，你会得到一个空的 `PeerCert`。在使用前务必做 `bool` 检查。

> 要设置 mTLS，参见 [T04. 配置 mTLS](../t04-mtls)。
