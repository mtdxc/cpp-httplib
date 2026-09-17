---
title: "W01. 实现一个 WebSocket 回声服务端与客户端"
order: 52
status: "draft"
---

WebSocket 是一个在客户端与服务端之间**双向**消息传递的协议。cpp-httplib 为两侧都提供了 API。我们从最简单的例子开始：一个回声服务端。

## 服务端：回声服务端

```cpp
#include <httplib.h>

int main() {
  httplib::Server svr;

  svr.WebSocket("/echo", [](const httplib::Request &req, httplib::ws::WebSocket &ws) {
    std::string msg;
    while (ws.is_open()) {
      auto result = ws.read(msg);
      if (result == httplib::ws::ReadResult::Fail) {
        break;
      }
      ws.send(msg); // echo back what we received
    }
  });

  svr.listen("0.0.0.0", 8080);
}
```

用 `svr.WebSocket()` 注册一个 WebSocket 处理函数。等到处理函数运行时，WebSocket 握手已经完成。在循环里，只需 `ws.read()` 和 `ws.send()` 就能得到一个可用的回声。

`read()` 的返回值是一个 `ReadResult` 枚举：

- `ReadResult::Text`：收到一个文本消息
- `ReadResult::Binary`：收到一个二进制消息
- `ReadResult::Fail`：出错，或连接已关闭
- `ReadResult::Timeout`：你用 `set_read_timeout()` 设置的读取超时无接收地到期；连接仍然打开。编译期默认超时会关闭连接并报告为 `Fail`——参见 [W06. 设置超时](../w06-websocket-timeouts)

## 客户端：与回声服务端对话

```cpp
#include <httplib.h>

int main() {
  httplib::ws::WebSocketClient cli("ws://localhost:8080/echo");
  if (!cli.connect()) {
    std::cerr << "failed to connect" << std::endl;
    return 1;
  }

  cli.send("Hello, WebSocket!");

  std::string msg;
  if (cli.read(msg) != httplib::ws::ReadResult::Fail) {
    std::cout << "received: " << msg << std::endl;
  }

  cli.close();
}
```

使用一个 `ws://`（普通）或 `wss://`（TLS）URL。调用 `connect()` 做握手，然后 `send()` 和 `read()` 的用法与服务端侧相同。

## 文本 vs. 二进制

`send()` 有两个重载让你选择帧类型。

```cpp
ws.send("Hello");                        // text frame
ws.send(binary_data, binary_data_size);  // binary frame
```

`std::string` 重载以**文本**发送；`const char*` + 大小的重载以**二进制**发送。有点微妙，但一旦知道就很直观。详情参见 [W04. 发送与接收二进制帧](../w04-websocket-binary)。

## 对线程池的影响

一个 WebSocket 处理函数会在连接的整个生命周期内占用它的工作线程——每连接一个线程。对于大量并发客户端，配置一个动态线程池。

```cpp
svr.new_task_queue = [] {
  return new httplib::ThreadPool(8, 128);
};
```

参见 [S21. 配置线程池](../s21-thread-pool)。

> **提示：** 要在 HTTPS 上运行 WebSocket，使用 `httplib::SSLServer` 而不是 `httplib::Server`——同样的 `WebSocket()` 处理函数照常可用。在客户端侧，使用一个 `wss://` URL。关于 CA 和客户端证书配置，参见 [W05. 为 wss:// 连接配置 TLS](../w05-websocket-tls)。
