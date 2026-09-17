---
title: "基础客户端"
order: 2
---

cpp-httplib 不只是服务器——它还自带一个完整的 HTTP 客户端。让我们用 `httplib::Client` 发送 GET 和 POST 请求。

## 准备测试服务器

要试用客户端，你需要一个能接受请求的服务器。保存下面的代码，然后像上一章那样编译并运行它。服务器的细节我们在下一章讲。

```cpp
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Server svr;

    svr.Get("/hi", [](const auto &, auto &res) {
        res.set_content("Hello!", "text/plain");
    });

    svr.Get("/search", [](const auto &req, auto &res) {
        auto q = req.get_param_value("q");
        res.set_content("Query: " + q, "text/plain");
    });

    svr.Post("/post", [](const auto &req, auto &res) {
        res.set_content(req.body, "text/plain");
    });

    svr.Post("/submit", [](const auto &req, auto &res) {
        std::string result;
        for (auto &[key, val] : req.params) {
            result += key + " = " + val + "\n";
        }
        res.set_content(result, "text/plain");
    });

    svr.Post("/upload", [](const auto &req, auto &res) {
        auto f = req.form.get_file("file");
        auto content = f.filename + " (" + std::to_string(f.content.size()) + " bytes)";
        res.set_content(content, "text/plain");
    });

    svr.Get("/users/:id", [](const auto &req, auto &res) {
        auto id = req.path_params.at("id");
        res.set_content("User ID: " + id, "text/plain");
    });

    svr.Get(R"(/files/(\d+))", [](const auto &req, auto &res) {
        auto id = req.matches[1];
        res.set_content("File ID: " + std::string(id), "text/plain");
    });

    std::cout << "Listening on port 8080..." << std::endl;
    svr.listen("0.0.0.0", 8080);
}
```

## GET 请求

服务器运行起来后，另开一个终端试一试。先从最简单的 GET 请求开始。

```cpp
#include "httplib.h"
#include <iostream>

int main() {
    httplib::Client cli("http://localhost:8080");

    auto res = cli.Get("/hi");
    if (res) {
        std::cout << res->status << std::endl;  // 200
        std::cout << res->body << std::endl;    // Hello!
    }
}
```

把服务器地址传给 `httplib::Client` 构造函数，然后调用 `Get()` 发送请求。你可以从返回的 `res` 中获取状态码和响应体。

等价的 `curl` 命令如下。

```sh
curl http://localhost:8080/hi
# Hello!
```

## 检查响应

响应除了状态码和响应体，还包含头部信息。

```cpp
auto res = cli.Get("/hi");
if (res) {
    // Status code
    std::cout << res->status << std::endl;  // 200

    // Body
    std::cout << res->body << std::endl;  // Hello!

    // Headers
    std::cout << res->get_header_value("Content-Type") << std::endl;  // text/plain
}
```

`res->body` 是 `std::string`，所以如果你想解析 JSON 响应，可以直接把它交给 [nlohmann/json](https://github.com/nlohmann/json) 这样的 JSON 库。

## 查询参数

给 GET 请求添加查询参数，可以直接写在 URL 里，也可以使用 `httplib::Params`。

```cpp
auto res = cli.Get("/search", httplib::Params{{"q", "cpp-httplib"}});
if (res) {
    std::cout << res->body << std::endl;  // Query: cpp-httplib
}
```

`httplib::Params` 会自动对特殊字符做 URL 编码。

```sh
curl "http://localhost:8080/search?q=cpp-httplib"
# Query: cpp-httplib
```

## 路径参数

当值直接嵌在 URL 路径里时，不需要任何特殊的客户端 API。把路径原样传给 `Get()` 即可。

```cpp
auto res = cli.Get("/users/42");
if (res) {
    std::cout << res->body << std::endl;  // User ID: 42
}
```

```sh
curl http://localhost:8080/users/42
# User ID: 42
```

测试服务器还有一个 `/files/(\d+)` 路由，用正则表达式只接受数字 ID。

```cpp
auto res = cli.Get("/files/42");
if (res) {
    std::cout << res->body << std::endl;  // File ID: 42
}
```

```sh
curl http://localhost:8080/files/42
# File ID: 42
```

传入 `/files/abc` 这样的非数字 ID，你会得到 404。其中的原理我们在下一章讲。

## 请求头

要添加自定义 HTTP 头，传入一个 `httplib::Headers` 对象。`Get()` 和 `Post()` 都支持。

```cpp
auto res = cli.Get("/hi", httplib::Headers{
    {"Authorization", "Bearer my-token"}
});
```

```sh
curl -H "Authorization: Bearer my-token" http://localhost:8080/hi
```

## POST 请求

我们来 POST 一段文本数据。把请求体作为 `Post()` 的第二个参数，Content-Type 作为第三个参数传入。

```cpp
auto res = cli.Post("/post", "Hello, Server!", "text/plain");
if (res) {
    std::cout << res->status << std::endl;  // 200
    std::cout << res->body << std::endl;    // Hello, Server!
}
```

测试服务器的 `/post` 端点会把请求体原样回显，所以返回的就是你发送的那个字符串。

```sh
curl -X POST -H "Content-Type: text/plain" -d "Hello, Server!" http://localhost:8080/post
# Hello, Server!
```

## 发送表单数据

你可以像 HTML 表单一样发送键值对。这用 `httplib::Params` 来实现。

```cpp
auto res = cli.Post("/submit", httplib::Params{
    {"name", "Alice"},
    {"age", "30"}
});
if (res) {
    std::cout << res->body << std::endl;
    // age = 30
    // name = Alice
}
```

这会以 `application/x-www-form-urlencoded` 格式发送数据。

```sh
curl -X POST -d "name=Alice&age=30" http://localhost:8080/submit
```

## POST 文件

要上传文件，用 `httplib::UploadFormDataItems` 以 multipart 表单数据发送。

```cpp
auto res = cli.Post("/upload", httplib::UploadFormDataItems{
    {"file", "Hello, File!", "hello.txt", "text/plain"}
});
if (res) {
    std::cout << res->body << std::endl;  // hello.txt (12 bytes)
}
```

`UploadFormDataItems` 的每个元素有四个字段：`{name, content, filename, content_type}`。

```sh
curl -F "file=Hello, File!;filename=hello.txt;type=text/plain" http://localhost:8080/upload
```

## 错误处理

网络通信可能失败——比如服务器无法访问。请务必检查 `res` 是否有效。

```cpp
httplib::Client cli("http://localhost:9999");  // Non-existent port
auto res = cli.Get("/hi");

if (!res) {
    // Connection error
    std::cout << "Error: " << httplib::to_string(res.error()) << std::endl;
    // Error: Connection
    return 1;
}

// If we reach here, we have a response
if (res->status != 200) {
    std::cout << "HTTP Error: " << res->status << std::endl;
    return 1;
}

std::cout << res->body << std::endl;
```

错误有两个层级。

- **连接错误**：客户端无法连上服务器。`res` 求值为 false，你可以调用 `res.error()` 查看具体原因。
- **HTTP 错误**：服务器返回了错误状态码（404、500 等）。`res` 求值为 true，但你需要检查 `res->status`。

## 下一步

现在你知道如何从客户端发送请求了。接下来让我们仔细看看服务器端。我们会深入讲解路由、路径参数等内容。

**下一章：**[基础服务器](../03-basic-server)
