---
title: "S22. 通过 Unix 域套接字通信"
order: 41
status: "draft"
---

当你只想与同一主机上的其他进程通信时，Unix 域套接字是一个不错的选择。它避免了 TCP 开销，并用文件系统权限做访问控制。本地 IPC 和跟在反向代理后面的服务是典型的用例。

## 服务端

```cpp
httplib::Server svr;
svr.set_address_family(AF_UNIX);

svr.Get("/", [](const auto &, auto &res) {
  res.set_content("hello from unix socket", "text/plain");
});

svr.listen("/tmp/httplib.sock", 80);
```

先调用 `set_address_family(AF_UNIX)`，然后把套接字文件路径作为第一个参数传给 `listen()`。端口号用不上，但签名要求它——传任意值即可。

## 客户端

```cpp
httplib::Client cli("/tmp/httplib.sock");
cli.set_address_family(AF_UNIX);

auto res = cli.Get("/");
if (res) {
  std::cout << res->body << std::endl;
}
```

把套接字文件路径传给 `Client` 构造函数并调用 `set_address_family(AF_UNIX)`。其他一切都像普通 HTTP 请求一样工作。

## 何时使用

- **在反向代理后面**：nginx 到后端的部署走 Unix 套接字比 TCP 更快，且回避了端口管理
- **仅本地的 API**：不应从外部访问的工具之间的 IPC
- **容器内 IPC**：同一 pod 或容器内的进程到进程通信
- **开发环境**：再也不用担心端口冲突

## 清理套接字文件

Unix 域套接字会在文件系统中创建一个真实文件。它在关闭时不会被移除，因此如有需要，在启动前删掉它。

```cpp
std::remove("/tmp/httplib.sock");
svr.listen("/tmp/httplib.sock", 80);
```

## 权限

你通过套接字文件的权限控制谁能连接。

```cpp
svr.listen("/tmp/httplib.sock", 80);
// from another process or thread
chmod("/tmp/httplib.sock", 0660); // owner and group only
```

> **警告：** 某些 Windows 版本支持 AF_UNIX，但实现和行为因平台而异。在跨环境用于生产之前，请彻底测试。
