---
title: "静态文件服务器"
order: 4
---

cpp-httplib 也能提供静态文件服务——HTML、CSS、图片，你想要的都行。不需要复杂配置，调用一次 `set_mount_point()` 就够。

## set_mount_point 基础

直接上手。`set_mount_point()` 把 URL 路径映射到本地目录。

```cpp
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Server svr;

    svr.set_mount_point("/", "./html");

    std::cout << "Listening on port 8080..." << std::endl;
    svr.listen("0.0.0.0", 8080);
}
```

第一个参数是 URL 挂载点，第二个是本地目录路径。这个例子中，对 `/` 的请求由 `./html` 目录提供服务。

我们来试试。先创建 `html` 目录，并添加一个 `index.html` 文件。

```sh
mkdir html
```

```html
<!DOCTYPE html>
<html>
<head><title>My Page</title></head>
<body>
    <h1>Hello from cpp-httplib!</h1>
    <p>This is a static file.</p>
</body>
</html>
```

编译并启动服务器。

```sh
g++ -std=c++17 -o server server.cpp -pthread
./server
```

在浏览器打开 `http://localhost:8080`。你应该能看到 `html/index.html` 的内容。访问 `http://localhost:8080/index.html` 返回同一个页面。

也可以用上一章的客户端代码或 `curl` 访问。

```cpp
httplib::Client cli("http://localhost:8080");
auto res = cli.Get("/");
if (res) {
    std::cout << res->body << std::endl;  // HTML is displayed
}
```

```sh
curl http://localhost:8080
```

## 多个挂载点

`set_mount_point()` 想调多少次都可以。每个 URL 路径可以对应自己的目录。

```cpp
svr.set_mount_point("/", "./public");
svr.set_mount_point("/assets", "./static/assets");
svr.set_mount_point("/docs", "./documentation");
```

对 `/assets/style.css` 的请求会返回 `./static/assets/style.css`。对 `/docs/guide.html` 的请求会返回 `./documentation/guide.html`。

## 与处理器结合

静态文件服务与上一章学的路由处理器可以并存。

```cpp
httplib::Server svr;

// API endpoint
svr.Get("/api/hello", [](const auto &, auto &res) {
    res.set_content(R"({"message":"Hello!"})", "application/json");
});

// Static file serving
svr.set_mount_point("/", "./public");

svr.listen("0.0.0.0", 8080);
```

处理器优先。`/api/hello` 由处理器响应。其他所有路径，服务器会在 `./public` 中查找文件。

## 添加响应头

把头部作为第三个参数传给 `set_mount_point()`，它们会被附加到每个静态文件响应上。非常适合做缓存控制。

```cpp
svr.set_mount_point("/", "./public", {
    {"Cache-Control", "max-age=3600"}
});
```

这样配置后，浏览器会把提供的文件缓存一小时。

## 静态文件服务器的 Dockerfile

cpp-httplib 仓库内置了一个为静态文件服务而构建的 `Dockerfile`。我们也在 Docker Hub 上发布了预构建镜像，一条命令即可启动。

```sh
> docker run -p 8080:80 -v ./my-site:/html yhirose4dockerhub/cpp-httplib-server
Serving HTTP on 0.0.0.0:80
Mount point: / -> ./html
Press Ctrl+C to shutdown gracefully...
192.168.65.1 - - [22/Feb/2026:12:00:00 +0000] "GET / HTTP/1.1" 200 256 "-" "Mozilla/5.0 ..."
192.168.65.1 - - [22/Feb/2026:12:00:00 +0000] "GET /style.css HTTP/1.1" 200 1024 "-" "Mozilla/5.0 ..."
192.168.65.1 - - [22/Feb/2026:12:00:01 +0000] "GET /favicon.ico HTTP/1.1" 404 152 "-" "Mozilla/5.0 ..."
```

`./my-site` 目录下的一切都会在 8080 端口上提供服务。访问日志的格式与 NGINX 相同，你能对发生了什么一目了然。

## 下一步

现在你能提供静态文件服务了。一个返回 HTML、CSS 和 JavaScript 的 Web 服务器——只用了这么一点代码就构建出来了。

接下来用 HTTPS 加密你的连接。先从配置 TLS 库开始。

**下一章：**[TLS 设置](../05-tls-setup)
