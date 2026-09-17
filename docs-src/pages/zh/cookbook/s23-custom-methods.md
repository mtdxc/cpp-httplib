---
title: "S23. 处理自定义 HTTP 方法"
order: 42
status: "draft"
---

服务端会用 `400 Bad Request` 拒绝它不认识的 HTTP 方法。要接受一个扩展方法，比如 RFC 4918 的 WebDAV 方法（`PROPFIND`、`PROPPATCH`、`MKCOL` 及同类）或 UPnP 的 `SUBSCRIBE`，用 `CustomRoute()` 注册一个处理函数。正是注册这个处理函数让服务端接受了该方法。

## 基本用法

```cpp
svr.CustomRoute("PROPFIND", "/dav/:id",
                [](const httplib::Request &req, httplib::Response &res) {
                  // The request body is available as usual
                  auto id = req.path_params.at("id");
                  res.status = httplib::StatusCode::MultiStatus_207;
                  res.set_content(build_multistatus(req.body), "application/xml");
                });
```

模式的用法与 `Get()` 完全相同。正则表达式和路径参数都可用。

## 用 OPTIONS 宣告你的方法

WebDAV 客户端在做任何事情之前，会用 `OPTIONS` 询问服务端的能力。cpp-httplib 既不生成 `DAV:` 头也不生成 `Allow`，所以你自己返回它们。忘了这一点，即使你的 `PROPFIND` 能用，客户端也会拒绝你。

```cpp
svr.Options("/dav/.*", [](const httplib::Request &req, httplib::Response &res) {
  res.set_header("DAV", "1");
  res.set_header("Allow", "OPTIONS, GET, HEAD, PROPFIND, PROPPATCH, MKCOL");
});
```

## 以流的方式读取请求体

有一个 content reader 重载，就像 `Post()` 上的那个。当你不想一次性把一个大的 XML 文档加载到内存时，用它。

```cpp
svr.CustomRoute("REPORT", "/dav/.*",
                [](const httplib::Request &req, httplib::Response &res,
                   const httplib::ContentReader &content_reader) {
                  content_reader([&](const char *data, size_t data_length) {
                    // Process it a chunk at a time
                    return true;
                  });
                  res.status = httplib::StatusCode::MultiStatus_207;
                });
```

## 需要记住的事

- 方法名必须是一个合法的 HTTP 方法 token（RFC 9110），且必须在调用 `listen()` 之前注册
- `GET`、`HEAD`、`POST`、`PUT`、`DELETE`、`CONNECT`、`OPTIONS`、`TRACE`、`PATCH` 和 `PRI` 不能在这里注册。用那些专用方法
- 被拒绝的注册会使 `is_valid()` 返回 `false` 且 `listen()` 失败，因此服务端绝不会带着一个永远不会运行的处理函数启动
- 静态文件服务和 WebSocket 升级仍仅限 `GET`/`HEAD`

> **提示：** cpp-httplib 能带你走到路由该方法这一步。如果你想称它为 WebDAV，生成 `207 Multi-Status` XML、解释 `Depth` 头以及管理锁都是你要自己实现的。协议本身在库之外。

> 关于注册处理函数的基础，参见 [S01. 注册 GET / POST / PUT / DELETE 处理函数](../s01-handlers)。
