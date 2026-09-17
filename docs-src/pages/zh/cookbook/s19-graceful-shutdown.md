---
title: "S19. 优雅关闭"
order: 38
status: "draft"
---

要停止服务端，调用 `Server::stop()`。即使仍有请求也可以安全调用，因此你可以把它接到 SIGINT 或 SIGTERM 上做优雅关闭。

## 基本用法

```cpp
httplib::Server svr;

svr.Get("/", [](const auto &, auto &res) { res.set_content("ok", "text/plain"); });

std::thread t([&] { svr.listen("0.0.0.0", 8080); });

// wait for input on the main thread, or whatever
std::cin.get();

svr.stop();
t.join();
```

`listen()` 会阻塞，所以典型的做法是：在后台线程上运行服务端，从主线程调用 `stop()`。`stop()` 后，`listen()` 会返回，你可以 `join()`。

## 收到信号时关闭

下面是在 SIGINT（Ctrl+C）或 SIGTERM 上停止服务端的方式。

```cpp
#include <csignal>

httplib::Server svr;

// global so the signal handler can reach it
httplib::Server *g_svr = nullptr;

int main() {
  svr.Get("/", [](const auto &, auto &res) { res.set_content("ok", "text/plain"); });

  g_svr = &svr;
  std::signal(SIGINT,  [](int) { if (g_svr) g_svr->stop(); });
  std::signal(SIGTERM, [](int) { if (g_svr) g_svr->stop(); });

  svr.listen("0.0.0.0", 8080);
  std::cout << "server stopped" << std::endl;
}
```

`stop()` 是线程安全且信号安全的——你可以从信号处理器调用它。即使 `listen()` 在主线程上运行，信号也能把它干净地拉出来。

## 在途请求会怎样

当你调用 `stop()` 时，新的连接会被拒绝，但已在处理的请求会**被允许完成**。一旦所有工作线程排空，`listen()` 返回。这就是它优雅之处。

> **警告：** 从调用 `stop()` 到 `listen()` 返回之间有一段等待——那是在途请求完成所需的时间。要强制执行一个超时，你需要在应用代码里自己加一个关闭定时器。
