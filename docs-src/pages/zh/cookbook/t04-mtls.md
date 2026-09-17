---
title: "T04. 配置 mTLS"
order: 46
status: "draft"
---

普通 TLS 只验证服务端证书。**mTLS**（双向 TLS）加上了另一个方向：客户端也出示一个证书，服务端验证它。它在零信任的 API 间流量和内部系统认证中很常见。

## 服务端

将用于验证客户端证书的 CA 作为第三个（以及第四个）参数传给 `SSLServer`。

```cpp
httplib::SSLServer svr(
  "server-cert.pem",    // server certificate
  "server-key.pem",     // server private key
  "client-ca.pem",      // CA that signs valid client certs
  nullptr               // CA directory (none)
);

svr.Get("/", [](const httplib::Request &req, httplib::Response &res) {
  res.set_content("authenticated", "text/plain");
});

svr.listen("0.0.0.0", 443);
```

这样一来，任何客户端证书未经 `client-ca.pem` 签名的连接都会在握手时被拒绝。等到处理函数运行时，客户端已经通过了认证。

## 用内存中的 PEM 配置

```cpp
httplib::SSLServer::PemMemory pem{};
pem.cert_pem = server_cert.data();
pem.cert_pem_len = server_cert.size();
pem.key_pem = server_key.data();
pem.key_pem_len = server_key.size();
pem.client_ca_pem = client_ca.data();
pem.client_ca_pem_len = client_ca.size();

httplib::SSLServer svr(pem);
```

当你从环境变量或密钥管理器加载证书时，这是一种干净的做法。

## 客户端

在客户端，将客户端证书和密钥传给 `SSLClient`。

```cpp
httplib::SSLClient cli("api.example.com", 443,
                       "client-cert.pem",
                       "client-key.pem");

auto res = cli.Get("/");
```

注意你用的是 `SSLClient` 而不是 `Client`。如果私钥有密码，把它作为第五个参数传入。

客户端侧也有同样的 `PemMemory` 结构体，让你能从内存中的 PEM 设置客户端证书。

```cpp
httplib::SSLClient::PemMemory pem{};
pem.cert_pem = client_cert.data();
pem.cert_pem_len = client_cert.size();
pem.key_pem = client_key.data();
pem.key_pem_len = client_key.size();

httplib::SSLClient cli("api.example.com", 443, pem);

auto res = cli.Get("/");
```

> 关于 WebSocket 客户端（`wss://`）上的 mTLS，参见 [W05. 为 wss:// 连接配置 TLS](../w05-websocket-tls)。

## 从处理函数中读取客户端信息

要在处理函数内看是哪个客户端连接的，使用 `req.peer_cert()`。详情见 [T05. 在服务端访问对端证书](../t05-peer-cert)。

## 用例

- **微服务到微服务的调用**：为每个服务颁发一个证书，用证书作为身份
- **IoT 设备管理**：向每个设备烧入一个证书，用它来把守 API 访问
- **内部 VPN 的一种替代**：在公开端点前放一个基于证书的认证，以便安全地访问内部资源

> **提示：** 颁发和吊销客户端证书比基于密码的认证需要更多运维工作。你需要要么搭一个内部 PKI，要么用一个基于 ACME 类工具的自动化流程。
