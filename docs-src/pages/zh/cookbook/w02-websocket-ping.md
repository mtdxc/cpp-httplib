---
title: "W02. 设置 WebSocket 心跳"
order: 53
status: "draft"
---

WebSocket 连接会长时间保持打开，而代理或负载均衡器有时会因为它“空闲”而将其断开。为防止这种情况，你周期性地发送 Ping 帧来保活连接。cpp-httplib 可以自动为你完成这件事。

## 服务端

```cpp
svr.set_websocket_ping_interval(30); // ping every 30 seconds

svr.WebSocket("/chat", [](const auto &req, auto &ws) {
  // ...
});
```

只需传入以秒为单位的间隔。该服务端接受的每一个 WebSocket 连接都会按这个间隔被 ping。

还有一个 `std::chrono` 重载。

```cpp
using namespace std::chrono_literals;
svr.set_websocket_ping_interval(30s);
```

## 客户端

客户端有相同的 API。

```cpp
httplib::ws::WebSocketClient cli("ws://localhost:8080/chat");
cli.set_websocket_ping_interval(30);
cli.connect();
```

在 `connect()` 之前调用它。

## 默认值

默认间隔由构建期宏 `CPPHTTPLIB_WEBSOCKET_PING_INTERVAL_SECOND` 设置。通常你不需要修改它，但如果要应对一个激进的代理，可以适当调小。

## 那 Pong 呢？

WebSocket 协议要求 Ping 帧要用 Pong 帧应答。cpp-httplib 会自动响应 Ping——你无需在应用代码里考虑它。

## 选择一个间隔

| 环境 | 建议 |
| --- | --- |
| 普通互联网 | 30–60s |
| 严格的代理（例如 AWS ALB） | 15–30s |
| 移动网络 | 60s+（太短会耗尽电量） |

太短会浪费带宽；太长则连接会被断开。作为一个经验法则，目标设为你与客户端之间任何环节的**空闲超时的约一半**。

> **警告：** 非常短的 ping 间隔会为每个连接派生后台工作并增加 CPU 使用率。对于连接数很多的服务器，请保持间隔适度。

## 检测无响应的对端

如果对端只是静默地死掉，光发送 ping 并不能告诉你任何信息——在另一端进程早已消失时，TCP 套接字可能仍然看起来是打开的。为了捕捉这种情况，启用 max-missed-pongs 检查：如果连续 N 次 ping 未收到应答，连接就会被关闭。

```cpp
cli.set_websocket_max_missed_pongs(2); // close after 2 consecutive unacked pings
```

服务端也有相同的 `set_websocket_max_missed_pongs()`。

在 30 秒的 ping 间隔和 `max_missed_pongs = 2` 下，一个死掉的对端会在大约 60 秒内被检测到，连接会以 `CloseStatus::GoingAway` 和原因 `"pong timeout"` 关闭。

每当 `read()` 消费一个入站的 Pong 帧时，计数器就会被重置，所以只有当你的代码在一个循环里主动调用 `read()` 时这才有效——而这也正是一个正常 WebSocket 客户端本就在做的事。

### 为什么默认值是 0

`max_missed_pongs` 默认为 `0`，意思是“绝不因为缺失 pong 而关闭连接”。Ping 仍会按心跳间隔发送，但它们的响应不会被检查。如果你想要无响应对端检测，就显式地把它设为 `1` 或更高。

在服务端侧，即使设为 `0`，一个死掉的连接也不会永远悬挂：当处理函数处于 `read()` 内部时，`CPPHTTPLIB_WEBSOCKET_SERVER_READ_TIMEOUT_SECOND`（默认 **300 秒 = 5 分钟**）充当一个兜底。客户端则没有自己的兜底——除非你设置读取超时，否则它会永远等待——因此在那里 `max_missed_pongs` 正是用来察觉一个无响应对端的东西。在任一侧，它也是你比那个 5 分钟兜底**更快**察觉对端的方式。

> 关于处理一个已关闭的连接，参见 [W03. 处理连接关闭](../w03-websocket-close)。
