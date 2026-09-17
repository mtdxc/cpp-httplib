---
title: "S20. 调优 Keep-Alive"
order: 39
status: "draft"
---

`httplib::Server` 会自动启用 HTTP/1.1 Keep-Alive。从客户端的视角看，连接被复用——因此他们无需在每个请求上都付出 TCP 握手的开销。当你需要调优这个行为时，有两个 setter。

## 你能配置什么

| API | 默认值 | 含义 |
| --- | --- | --- |
| `set_keep_alive_max_count` | 100 | 单条连接上服务的最大请求数 |
| `set_keep_alive_timeout` | 5秒 | 一条空闲连接在关闭前保持多长时间 |

## 基本用法

```cpp
httplib::Server svr;

svr.set_keep_alive_max_count(20);
svr.set_keep_alive_timeout(10); // 10 seconds

svr.listen("0.0.0.0", 8080);
```

`set_keep_alive_timeout()` 也有一个 `std::chrono` 重载。

```cpp
using namespace std::chrono_literals;
svr.set_keep_alive_timeout(10s);
```

## 调优思路

**太多空闲连接在吞噬资源**  
缩短超时，使空闲连接断开并释放它们的工作线程。

```cpp
svr.set_keep_alive_timeout(2s);
```

**API 被高频命中，你想要最大复用**  
抬高每连接请求上限能改善基准测试数字。

```cpp
svr.set_keep_alive_max_count(1000);
```

**从不复用连接**  
设置 `set_keep_alive_max_count(1)`，每个请求都得到自己的连接。主要只在调试或兼容性测试时有用。

## 与线程池的关系

一条 Keep-Alive 连接会在其整个生命周期内占用一个工作线程。如果 `连接数 × 并发请求数` 超过线程池大小，新请求就会等待。关于线程数，参见 [S21. 配置线程池](../s21-thread-pool)。

> **提示：** 关于客户端侧，参见 [C14. 理解连接复用与 Keep-Alive 行为](../c14-keep-alive)。即使服务端在超时时关闭了连接，客户端也会自动重连。
