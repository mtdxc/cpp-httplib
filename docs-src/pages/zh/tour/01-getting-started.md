---
title: "入门"
order: 1
---

开始使用 cpp-httplib，你只需要 `httplib.h` 和一个 C++ 编译器。让我们下载文件，运行一个 Hello World 服务器。

## 获取 httplib.h

你可以直接从 GitHub 下载。请始终使用最新版本。

```sh
curl -LO https://github.com/yhirose/cpp-httplib/raw/refs/tags/latest/httplib.h
```

把下载好的 `httplib.h` 放进你的项目目录中，就可开始了。

## 配置编译器

| 操作系统 | 开发环境 | 配置方式 |
| -- | ----------------------- | ----- |
| macOS | Apple Clang | Xcode Command Line Tools（`xcode-select --install`） |
| Ubuntu | clang++ 或 g++ | `apt install clang` 或 `apt install g++` |
| Windows | MSVC | Visual Studio 2022 或更新版本（安装时勾选 C++ 组件） |

## Hello World 服务器

把下面的代码保存为 `server.cpp`。

```cpp
#include "httplib.h"

int main() {
    httplib::Server svr;

    svr.Get("/", [](const httplib::Request&, httplib::Response& res) {
        res.set_content("Hello, World!", "text/plain");
    });

    svr.listen("0.0.0.0", 8080);
}
```

短短几行，你就拥有了一个能响应 HTTP 请求的服务器。

## 编译与运行

本教程的示例代码使用 C++17 编写，以让代码更清晰简洁。cpp-httplib 本身用 C++11 也能编译。

```sh
# macOS
clang++ -std=c++17 -o server server.cpp

# Linux
# `-pthread`: cpp-httplib uses threads internally
clang++ -std=c++17 -pthread -o server server.cpp

# Windows (Developer Command Prompt)
# `/EHsc`: Enable C++ exception handling
cl /EHsc /std:c++17 server.cpp
```

编译通过后，运行它。

```sh
# macOS / Linux
./server

# Windows
server.exe
```

在浏览器中打开 `http://localhost:8080`。如果看到 "Hello, World!"，一切就绪。

也可以用 `curl` 验证。

```sh
curl http://localhost:8080/
# Hello, World!
```

要停止服务器，在终端按 `Ctrl+C`。

## 下一步

现在你已经掌握了运行服务器的基础知识。接下来看看客户端。cpp-httplib 同样自带 HTTP 客户端功能。

**下一章：**[基础客户端](../02-basic-client)
