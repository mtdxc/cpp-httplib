---
title: "基础服务器"
order: 3
---

上一章我们从客户端向测试服务器发送了请求。现在我们来梳理那个服务器究竟是如何工作的。

## 启动服务器

注册好路由后，调用 `svr.listen()` 启动服务器。

```cpp
svr.listen("0.0.0.0", 8080);
```

第一个参数是主机，第二个是端口。`"0.0.0.0"` 表示监听所有网络接口。如果只想接受来自本机的连接，用 `"127.0.0.1"`。

`listen()` 是阻塞调用。服务器停止前它不会返回。服务器会一直运行，直到你在终端按 `Ctrl+C`，或从另一个线程调用 `svr.stop()`。

## 路由

路由是任何服务器的核心。它告诉 cpp-httplib：当这个 URL、这种 HTTP 方法的请求进来时，执行这段代码。

```cpp
httplib::Server svr;

svr.Get("/hi", [](const httplib::Request &req, httplib::Response &res) {
    res.set_content("Hello!", "text/plain");
});
```

`svr.Get()` 为 GET 请求注册一个处理器。第一个参数是路径，第二个是处理函数。当 `/hi` 收到 GET 请求时，你的 lambda 就会执行。

每个 HTTP 方法都有对应的方法。

```cpp
svr.Get("/path",    handler);  // GET
svr.Post("/path",   handler);  // POST
svr.Put("/path",    handler);  // PUT
svr.Delete("/path", handler);  // DELETE
```

处理器的签名是 `(const httplib::Request &req, httplib::Response &res)`。可以用 `auto` 让它更简短。

```cpp
svr.Get("/hi", [](const auto &req, auto &res) {
    res.set_content("Hello!", "text/plain");
});
```

只有路径匹配时处理器才会运行。未注册路径的请求会自动返回 404。

## Request 对象

第一个参数 `req` 提供了客户端发送的一切信息。

### 请求体

`req.body` 以 `std::string` 保存请求体。

```cpp
svr.Post("/post", [](const auto &req, auto &res) {
    // Echo the body back to the client
    res.set_content(req.body, "text/plain");
});
```

### 请求头

用 `req.get_header_value()` 读取请求头。

```cpp
svr.Get("/check", [](const auto &req, auto &res) {
    auto auth = req.get_header_value("Authorization");
    res.set_content("Auth: " + auth, "text/plain");
});
```

### 查询参数与表单数据

`req.get_param_value()` 按名称获取参数。GET 查询参数和 POST 表单数据都适用。

```cpp
svr.Get("/search", [](const auto &req, auto &res) {
    auto q = req.get_param_value("q");
    res.set_content("Query: " + q, "text/plain");
});
```

对 `/search?q=cpp-httplib` 的请求会把 `q` 取为 `"cpp-httplib"`。

要遍历所有参数，使用 `req.params`。

```cpp
svr.Post("/submit", [](const auto &req, auto &res) {
    std::string result;
    for (auto &[key, val] : req.params) {
        result += key + " = " + val + "\n";
    }
    res.set_content(result, "text/plain");
});
```

### 文件上传

通过 multipart 表单数据上传的文件可以用 `req.form.get_file()` 获取。

```cpp
svr.Post("/upload", [](const auto &req, auto &res) {
    auto f = req.form.get_file("file");
    auto content = f.filename + " (" + std::to_string(f.content.size()) + " bytes)";
    res.set_content(content, "text/plain");
});
```

`f.filename` 给出文件名，`f.content` 给出文件数据。

## 路径参数

有时你想把 URL 的一部分作为变量捕获——例如 `/users/42` 中的 `42`。用 `:param` 语法即可。

```cpp
svr.Get("/users/:id", [](const auto &req, auto &res) {
    auto id = req.path_params.at("id");
    res.set_content("User ID: " + id, "text/plain");
});
```

对 `/users/42` 的请求，`req.path_params.at("id")` 得到 `"42"`。`/users/100` 得到 `"100"`。

可以一次捕获多个段。

```cpp
svr.Get("/users/:user_id/posts/:post_id", [](const auto &req, auto &res) {
    auto user_id = req.path_params.at("user_id");
    auto post_id = req.path_params.at("post_id");
    res.set_content("User: " + user_id + ", Post: " + post_id, "text/plain");
});
```

### 正则模式

也可以在路径里直接写正则表达式，而不必用 `:param`。捕获组的值可通过 `req.matches` 获取，它是一个 `std::smatch`。

```cpp
// Only accept numeric IDs
svr.Get(R"(/files/(\d+))", [](const auto &req, auto &res) {
    auto id = req.matches[1];  // First capture group
    res.set_content("File ID: " + std::string(id), "text/plain");
});
```

`/files/42` 能匹配，但 `/files/abc` 不能。当你想限制接受的值时，这很好用。

## 构建响应

第二个参数 `res` 就是你向客户端回传数据的方式。

### 响应体与 Content-Type

`res.set_content()` 设置响应体和 Content-Type。返回 200 响应只需这些。

```cpp
svr.Get("/hi", [](const auto &req, auto &res) {
    res.set_content("Hello!", "text/plain");
});
```

### 状态码

要返回不同的状态码，给 `res.status` 赋值。

```cpp
svr.Get("/not-found", [](const auto &req, auto &res) {
    res.status = 404;
    res.set_content("Not found", "text/plain");
});
```

### 响应头

用 `res.set_header()` 添加响应头。

```cpp
svr.Get("/with-header", [](const auto &req, auto &res) {
    res.set_header("X-Custom", "my-value");
    res.set_content("Hello!", "text/plain");
});
```

## 逐行解读测试服务器

现在用我们学到的知识，来读一遍上一章的测试服务器。

### GET /hi

```cpp
svr.Get("/hi", [](const auto &, auto &res) {
    res.set_content("Hello!", "text/plain");
});
```

最简单的处理器。我们不需要请求里的任何信息，所以 `req` 参数不命名。它只返回 `"Hello!"`。

### GET /search

```cpp
svr.Get("/search", [](const auto &req, auto &res) {
    auto q = req.get_param_value("q");
    res.set_content("Query: " + q, "text/plain");
});
```

`req.get_param_value("q")` 取出查询参数 `q`。对 `/search?q=cpp-httplib` 的请求返回 `"Query: cpp-httplib"`。

### POST /post

```cpp
svr.Post("/post", [](const auto &req, auto &res) {
    res.set_content(req.body, "text/plain");
});
```

一个回显服务器。客户端发送什么请求体，`req.body` 就存着什么，我们原样返回。

### POST /submit

```cpp
svr.Post("/submit", [](const auto &req, auto &res) {
    std::string result;
    for (auto &[key, val] : req.params) {
        result += key + " = " + val + "\n";
    }
    res.set_content(result, "text/plain");
});
```

用结构化绑定（`auto &[key, val]`）遍历 `req.params` 中的表单数据，解开每个键值对。

### POST /upload

```cpp
svr.Post("/upload", [](const auto &req, auto &res) {
    auto f = req.form.get_file("file");
    auto content = f.filename + " (" + std::to_string(f.content.size()) + " bytes)";
    res.set_content(content, "text/plain");
});
```

接收通过 multipart 表单数据上传的文件。`req.form.get_file("file")` 取出名为 `"file"` 的字段，我们返回文件名和大小。

### GET /users/:id

```cpp
svr.Get("/users/:id", [](const auto &req, auto &res) {
    auto id = req.path_params.at("id");
    res.set_content("User ID: " + id, "text/plain");
});
```

`:id` 是路径参数。`req.path_params.at("id")` 获取它的值。`/users/42` 得到 `"42"`，`/users/alice` 得到 `"alice"`。

### GET /files/(\d+)

```cpp
svr.Get(R"(/files/(\d+))", [](const auto &req, auto &res) {
    auto id = req.matches[1];
    res.set_content("File ID: " + std::string(id), "text/plain");
});
```

正则 `(\d+)` 只匹配数字 ID。`/files/42` 会进入这个处理器，但 `/files/abc` 返回 404。`req.matches[1]` 获取第一个捕获组。

## 下一步

现在你已经掌握服务器工作的全貌。路由、读取请求、构建响应——这些就足够构建一个真正的 API 服务器了。

接下来看看如何提供静态文件。我们会构建一个能返回 HTML 和 CSS 的服务器。

**下一章：**[静态文件服务器](../04-static-file-server)
