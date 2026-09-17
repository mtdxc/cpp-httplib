---
title: "C12. 设置超时"
order: 12
status: "draft"
---

客户端有三类超时，每一类都独立设置。

| 类型 | API | 默认值 | 含义 |
| --- | --- | --- | --- |
| 连接 | `set_connection_timeout` | 300秒 | 等待 TCP 连接建立的时间 |
| 读取 | `set_read_timeout` | 300秒 | 接收响应时等待单次 `recv` 的时间 |
| 写入 | `set_write_timeout` | 5秒 | 发送请求时等待单次 `send` 的时间 |

## 基本用法

```cpp
httplib::Client cli("http://localhost:8080");

cli.set_connection_timeout(5, 0);  // 5 seconds
cli.set_read_timeout(10, 0);       // 10 seconds
cli.set_write_timeout(10, 0);      // 10 seconds

auto res = cli.Get("/api/data");
```

将秒和微秒作为两个参数传入。如果你不需要亚秒部分，可以省略第二个参数。

## 使用 `std::chrono`

还有一个直接接受 `std::chrono` 时长的重载。它更易读——推荐。

```cpp
using namespace std::chrono_literals;

cli.set_connection_timeout(5s);
cli.set_read_timeout(10s);
cli.set_write_timeout(500ms);
```

## 小心长达 300 秒的默认值

连接和读取超时默认为 **300 秒（5 分钟）**。如果服务端挂起，默认你会等上五分钟。通常更短的值是更好的选择。

```cpp
cli.set_connection_timeout(3s);
cli.set_read_timeout(10s);
```

> **警告：** 读取超时只覆盖单次接收调用——而不是整个请求。如果在大型下载中数据持续徐入，请求可能花上半个小时也从未触发超时。要限制总请求耗时，使用 [C13. 设置整体超时](../c13-max-timeout)。

> 关于 WebSocket 客户端超时，参见 [W06. 设置超时](../w06-websocket-timeouts)。
