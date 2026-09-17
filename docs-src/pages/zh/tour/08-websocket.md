---
title: "WebSocket"
order: 8
---

cpp-httplib 也支持 WebSocket。与 HTTP 的请求/响应不同，WebSocket 让服务器和客户端可以双向交换消息。非常适合聊天应用和实时通知。

我们马上来实现一个回显服务器和客户端。

## 回显服务器

下面这个回显服务器会把收到的消息原样发回。

```cpp
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Server svr;

    svr.WebSocket("/ws", [](const httplib::Request &, httplib::ws::WebSocket &ws) {
        std::string msg;
        while (ws.read(msg)) {
            ws.send(msg);  // Send back the received message as-is
        }
    });

    std::cout << "Listening on port 8080..." << std::endl;
    svr.listen("0.0.0.0", 8080);
}
```

用 `svr.WebSocket()` 注册一个 WebSocket 处理器。它和第 3 章的 `svr.Get()`、`svr.Post()` 用法一样。

在处理器里，`ws.read(msg)` 等待一条消息。连接关闭时，`read()` 返回 `false`，循环退出。`ws.send(msg)` 把消息发回。

## 从客户端连接

我们用 `httplib::ws::WebSocketClient` 连接服务器。

```cpp
#include "httplib.h"
#include <iostream>

int main() {
    httplib::ws::WebSocketClient client("ws://localhost:8080/ws");

    if (!client.connect()) {
        std::cout << "Connection failed" << std::endl;
        return 1;
    }

    // Send a message
    client.send("Hello, WebSocket!");

    // Receive a response from the server
    std::string msg;
    if (client.read(msg)) {
        std::cout << msg << std::endl;  // Hello, WebSocket!
    }

    client.close();
}
```

向构造函数传入 `ws://host:port/path` 格式的 URL。调用 `connect()` 建立连接，然后用 `send()` 和 `read()` 交换消息。

## 文本与二进制

WebSocket 有两种消息类型：文本和二进制。你可以通过 `read()` 的返回值区分它们。

```cpp
svr.WebSocket("/ws", [](const httplib::Request &, httplib::ws::WebSocket &ws) {
    std::string msg;
    httplib::ws::ReadResult ret;
    while ((ret = ws.read(msg))) {
        if (ret == httplib::ws::Binary) {
            ws.send(msg.data(), msg.size());  // Send as binary
        } else {
            ws.send(msg);  // Send as text
        }
    }
});
```

- `ws.send(const std::string &)` — 以文本消息发送
- `ws.send(const char *, size_t)` — 以二进制消息发送

客户端侧的 API 相同。

## 访问请求信息

你可以从处理器的第一个参数 `req` 读取握手阶段的 HTTP 请求信息。在检查认证 token 时很有用。

```cpp
svr.WebSocket("/ws", [](const httplib::Request &req, httplib::ws::WebSocket &ws) {
    auto token = req.get_header_value("Authorization");
    if (token.empty()) {
        ws.close(httplib::ws::CloseStatus::PolicyViolation, "unauthorized");
        return;
    }

    std::string msg;
    while (ws.read(msg)) {
        ws.send(msg);
    }
});
```

## 使用 WSS

也支持基于 HTTPS 的 WebSocket（WSS）。服务器端，只需在 `httplib::SSLServer` 上注册 WebSocket 处理器。

```cpp
httplib::SSLServer svr("cert.pem", "key.pem");

svr.WebSocket("/ws", [](const httplib::Request &, httplib::ws::WebSocket &ws) {
    std::string msg;
    while (ws.read(msg)) {
        ws.send(msg);
    }
});

svr.listen("0.0.0.0", 8443);
```

客户端使用 `wss://` 协议。

```cpp
httplib::ws::WebSocketClient client("wss://localhost:8443/ws");
```

## 下一步

现在你掌握了 WebSocket 的基础。向导到此结束。

下一页会总结向导中未涵盖的功能。

**下一章：**[下一步](../09-whats-next)
