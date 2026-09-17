---
title: "W06. 设置超时"
order: 57
status: "draft"
---

`ws::WebSocketClient` 拥有与 `Client` 相同的三种超时，含义也相同。

| 类型 | API | 默认值 |
| --- | --- | --- |
| 连接 | `set_connection_timeout` | 300s |
| 读取 | `set_read_timeout` | 无——永远等待（`CPPHTTPLIB_WEBSOCKET_CLIENT_READ_TIMEOUT_SECOND`） |
| 写入 | `set_write_timeout` | 5s |

## 基本用法

```cpp
httplib::ws::WebSocketClient ws("ws://localhost:8080/ws");

ws.set_connection_timeout(5, 0);  // 5 seconds
ws.set_read_timeout(30, 0);       // 30 seconds
ws.set_write_timeout(10, 0);      // 10 seconds

if (ws.connect()) {
  ws.send("hello");
}
```

在调用 `connect()` 之前设置连接和写入超时。读取超时可以随时修改——在一个已打开的连接上设置它，会在下一次 `read()` 时生效。

## 使用 `std::chrono`

与 `Client` 一样，有一个直接接受 `std::chrono` 时长的重载。

```cpp
using namespace std::chrono_literals;

ws.set_connection_timeout(5s);
ws.set_read_timeout(30s);
ws.set_write_timeout(10s);
```

## 读取超时的含义

`set_read_timeout()` 作用于单次 `read()` 调用。如果在那个时间内没有消息到达，`read()` 返回 `ReadResult::Timeout`：**连接仍然打开**，什么也没被消费，因此你可以在其上发送并再次读取。这正是它与 `ReadResult::Fail` 的区别，后者意味着连接已经没了。

这就是为什么一个线程能在读写两个方向上处理一个连接：

```cpp
using namespace std::chrono_literals;

ws.set_read_timeout(100ms);
std::string msg;
while (ws.is_open()) {
  auto r = ws.read(msg);
  if (r == httplib::ws::Timeout) {
    flush_outgoing(ws);  // nothing arrived — send whatever is queued
    continue;
  }
  if (r == httplib::ws::Fail) { break; }
  handle(msg);
}
```

没有读取超时时，`read()` 会阻塞直到消息到达，持有连接的线程永远轮不到它的写入。

关于 `Timeout` 有两点需要知道：

- 它不会碰 `msg`，而且它非零。所以一旦设置了读取超时，`while (ws.read(msg))` 就不能用了——循环会带着仍留在 `msg` 中的*上一条*消息继续运行。
- 它只在消息边界上报告。如果超时在一个分片消息进到一半时到期，该消息无法恢复，`read()` 会返回 `Fail`。

对于那些长空闲期很正常的连接——例如等待通知——要么保持读取超时未设置，要么把 `Timeout` 当作它本就是一个空操作并继续循环。

## 在服务端侧

处理函数的 `ws::WebSocket` 也有 `set_read_timeout()`，而上面的模式就是一个处理函数如何在连接之间转发而不是停在 `read()` 里。

服务端的默认值是 300s（`CPPHTTPLIB_WEBSOCKET_SERVER_READ_TIMEOUT_SECOND`）而不是“永远”：它是一个兜底，用于从一个已静默的对端那里回收工作线程，因为一个 WebSocket 处理函数会在连接的整个生命周期内占用它的工作线程。

因为它是一个兜底而非处理函数主动要求的，它不会以 `Timeout` 的形式出现。当它到期时，`read()` 会返回 `Fail` 并关闭连接，因此一个写成 `while (ws.read(msg))` 的处理函数会以它一贯的方式结束。只有处理函数自己用 `set_read_timeout()` 设置的超时才会以 `Timeout` 返回。

> 通过 Ping/Pong 进行无响应对端检测是一个单独的机制。详情参见 [W02. 设置 WebSocket 心跳](../w02-websocket-ping)。

## 这与 `Client` 有何不同

关于 `Client` 的超时配置，参见 [C12. 设置超时](../c12-timeouts)。行为和 API 几乎完全相同，但 `WebSocketClient` 没有相当于 `set_max_timeout()` 的东西来限制整个请求——一旦连接，只要你继续调用 `read()`，连接就会一直保持打开。
