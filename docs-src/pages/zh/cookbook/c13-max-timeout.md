---
title: "C13. 设置整体超时"
order: 13
status: "draft"
---

[C12. 设置超时](../c12-timeouts) 中的三个超时都只作用于单次 `send` 或 `recv` 调用。要限制一个请求总共能花多长时间，使用 `set_max_timeout()`。

## 基本用法

```cpp
httplib::Client cli("http://localhost:8080");

cli.set_max_timeout(5000); // 5 seconds (in milliseconds)

auto res = cli.Get("/slow-endpoint");
```

该值以毫秒为单位。连接、发送、接收加起来——如果整个请求超过该限制就会被中止。

## 使用 `std::chrono`

也有一个接受 `std::chrono` 时长的重载。

```cpp
using namespace std::chrono_literals;
cli.set_max_timeout(5s);
```

## 何时用哪个

`set_read_timeout` 在一段时间没有数据到达时触发。如果数据持续一点一点徐入，它永远不会触发。一个每秒只发送一个字节的端点，无论你把 `set_read_timeout` 设多短，它都会失效。

`set_max_timeout` 限制的是已耗时，所以它能干净地处理那些情况。它很适合用于调用外部 API，或任何你不想让用户无限等待的地方。

```cpp
cli.set_connection_timeout(3s);
cli.set_read_timeout(10s);
cli.set_max_timeout(30s); // abort if the whole request takes over 30s
```

> **提示：** `set_max_timeout()` 与常规超时一起协作。短暂的停顿由 `set_read_timeout` 捕获；长时间运行的请求由 `set_max_timeout` 封顶。两者一起用才能构成安全网。
