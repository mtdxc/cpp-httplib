---
title: "S17. 绑定到任意可用端口"
order: 36
status: "draft"
---

搭建一个测试服务端经常会撞上端口冲突。用 `bind_to_any_port()`，你让操作系统选一个空闲端口，然后读回它给你的到底是哪个。

## 基本用法

```cpp
httplib::Server svr;

svr.Get("/", [](const auto &req, auto &res) {
  res.set_content("hello", "text/plain");
});

int port = svr.bind_to_any_port("0.0.0.0");
std::cout << "listening on port " << port << std::endl;

svr.listen_after_bind();
```

`bind_to_any_port()` 等价于把 `0` 作为端口传入——操作系统会分配一个空闲的。返回值是实际使用的端口。

之后，调用 `listen_after_bind()` 开始接受连接。这里你不能把 bind 和 listen 合并成一个调用，所以分两步做。

## 在测试中很有用

这种写法非常适合那种启动一个服务端再去命中它的测试。

```cpp
httplib::Server svr;
svr.Get("/ping", [](const auto &, auto &res) { res.set_content("pong", "text/plain"); });

int port = svr.bind_to_any_port("127.0.0.1");
std::thread t([&] { svr.listen_after_bind(); });

// run the test while the server is up on another thread
httplib::Client cli("127.0.0.1", port);
auto res = cli.Get("/ping");
assert(res && res->body == "pong");

svr.stop();
t.join();
```

因为端口是在运行时分配的，并行的测试运行不会冲突。

> **提示：** `bind_to_any_port()` 失败时返回 `-1`（权限错误、没有可用端口等）。务必检查返回值。

> 要停止服务端，参见 [S19. 优雅关闭](../s19-graceful-shutdown)。
