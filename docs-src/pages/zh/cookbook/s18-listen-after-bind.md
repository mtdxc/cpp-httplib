---
title: "S18. 用 listen_after_bind 控制启动顺序"
order: 37
status: "draft"
---

通常 `svr.listen("0.0.0.0", 8080)` 一口气处理 bind 和 listen。当你需要在两者之间做点事情时，把它们拆成两个调用。

## 分开 bind 和 listen

```cpp
httplib::Server svr;

svr.Get("/", [](const auto &, auto &res) { res.set_content("ok", "text/plain"); });

if (!svr.bind_to_port("0.0.0.0", 8080)) {
  std::cerr << "bind failed" << std::endl;
  return 1;
}

// bind is done here. accept hasn't started yet.
drop_privileges();
signal_ready_to_parent_process();

svr.listen_after_bind(); // start the accept loop
```

`bind_to_port()` 保留端口；`listen_after_bind()` 才真正开始接受。拆分它们在两步之间给了你一个窗口。

## 常见用例

**降权**：绑定到 1024 以下的端口需要 root。以 root 绑定，降到普通用户，之后所有的请求处理都以降低的权限运行。

```cpp
svr.bind_to_port("0.0.0.0", 80);
drop_privileges();
svr.listen_after_bind();
```

**启动通知**：在开始接受连接前告诉父进程或 systemd “我准备好了”。

**测试同步**：在测试中，你可以可靠地捕获“服务端已绑定的时刻”，并在那之后启动客户端。

## 检查返回值

`bind_to_port()` 失败时返回 `false`，例如当你没有权限绑定到该端口时。务必检查它。

```cpp
if (!svr.bind_to_port("0.0.0.0", 8080)) {
  std::cerr << "bind failed" << std::endl;
  return 1;
}
```

`listen_after_bind()` 会阻塞直到服务端停止，并在干净关闭时返回 `true`。

## 检测已被占用的端口

在默认设置下，你实际上可以绑定到另一个服务端已在使用的端口。这是因为 cpp-httplib 在服务端 socket 上设置了 `SO_REUSEPORT`（Linux、macOS）或 `SO_REUSEADDR`（Windows）。重启的服务端能立即重新绑定。反面则是，同一个端口上的第二个服务端会无错启动，而连接会在两者之间被分摊。

要让 `bind_to_port()` 在端口被占用时失败，用 `set_socket_options()` 替换 socket 选项。

```cpp
svr.set_socket_options([](socket_t sock) {
#ifdef _WIN32
  httplib::set_socket_opt(sock, SOL_SOCKET, SO_EXCLUSIVEADDRUSE, 1);
#else
  httplib::set_socket_opt(sock, SOL_SOCKET, SO_REUSEADDR, 1);
#endif
});

if (!svr.bind_to_port("0.0.0.0", 8080)) {
  std::cerr << "port already in use" << std::endl;
  return 1;
}
```

`set_socket_options()` 会完全替换默认值。在 Linux 和 macOS 上设置 `SO_REUSEADDR` 保留了“重启的服务端能立即重新绑定”的行为。

> **提示：** 在 Windows 上，单靠 `SO_REUSEADDR` 不够。两个都设置了它的 socket 可以绑定到同一端口，因此改用 `SO_EXCLUSIVEADDRUSE`。

> **提示：** 要自动选一个空闲端口，参见 [S17. 绑定到任意可用端口](../s17-bind-any-port)。在其底层，那就是 `bind_to_any_port()` + `listen_after_bind()`。
