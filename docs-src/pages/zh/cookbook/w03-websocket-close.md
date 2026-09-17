---
title: "W03. 处理连接关闭"
order: 54
status: "draft"
---

当任意一侧显式关闭 WebSocket，或网络掉线时，连接就会结束。干净地处理关闭，你的清理和重连逻辑就会保持整洁。

## 检测一个已关闭的连接

当 `ws.read()` 返回 `ReadResult::Fail` 时，连接已经没了——要么是干净关闭，要么带着错误。跳出循环，处理函数就会结束。

```cpp
svr.WebSocket("/chat", [](const httplib::Request &req, httplib::ws::WebSocket &ws) {
  std::string msg;
  while (ws.is_open()) {
    auto result = ws.read(msg);
    if (result == httplib::ws::ReadResult::Fail) {
      std::cout << "disconnected" << std::endl;
      break;
    }
    handle_message(ws, msg);
  }

  // cleanup runs once we're out of the loop
  cleanup_user_session(req);
});
```

你也可以检查 `ws.is_open()`——它是同一个信号从另一个角度的体现。

## 从服务端侧关闭

要显式关闭，调用 `close()`。

```cpp
ws.close(httplib::ws::CloseStatus::Normal, "bye");
```

第一个参数是关闭状态；第二个是可选的原因。常见的 `CloseStatus` 值：

| 值 | 含义 |
| --- | --- |
| `Normal` (1000) | 正常关闭 |
| `GoingAway` (1001) | 服务端正在关闭 |
| `ProtocolError` (1002) | 检测到协议违规 |
| `UnsupportedData` (1003) | 收到了无法处理的数据 |
| `PolicyViolation` (1008) | 违反了某项策略 |
| `MessageTooBig` (1009) | 消息过大 |
| `InternalError` (1011) | 服务端错误 |

## 从客户端侧关闭

客户端 API 完全相同。

```cpp
cli.close(httplib::ws::CloseStatus::Normal);
```

销毁客户端也会关闭连接，但显式调用 `close()` 会让意图更清晰。

## 优雅关闭

要通知仍在进行的客户端服务端即将关闭，使用 `GoingAway`。

```cpp
ws.close(httplib::ws::CloseStatus::GoingAway, "server restarting");
```

客户端可以检查那个状态，并决定是否重连。

## 示例：一个带退出的小聊天

```cpp
svr.WebSocket("/chat", [](const auto &req, auto &ws) {
  std::string msg;
  while (ws.is_open()) {
    if (ws.read(msg) == httplib::ws::ReadResult::Fail) break;

    if (msg == "/quit") {
      ws.send("goodbye");
      ws.close(httplib::ws::CloseStatus::Normal, "user quit");
      break;
    }

    ws.send("echo: " + msg);
  }
});
```

> **提示：** 在突然的网络掉线上，`read()` 会返回 `Fail`，根本没有机会调用 `close()`。把你的清理放在处理函数的末尾，这样两条路径——干净关闭和突然断开——都会汇入同一个地方。
