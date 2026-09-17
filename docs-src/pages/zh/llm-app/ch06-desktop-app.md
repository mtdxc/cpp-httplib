---
title: "6. 用 WebView 变成桌面应用"
order: 6

---

第 5 章，我们完成了一个能在浏览器中使用的翻译应用。但每次都得启动服务器、在浏览器里打开网址……如果能像普通应用一样双击就启动，岂不更好？

本章我们做两件事：

1. **WebView 集成** — 用 [webview/webview](https://github.com/webview/webview) 把它变成无需浏览器即可运行的桌面应用
2. **单二进制打包** — 用 [cpp-embedlib](https://github.com/yhirose/cpp-embedlib) 把 HTML/CSS/JS 嵌入二进制，使分发产物成为单文件

完成后，你只需运行 `./translate-app` 就能打开一个窗口开始翻译。

![Desktop App](../app.png#large-center)

模型会在首次启动时自动下载，你交给用户的，只有那个单二进制文件。

## 6.1 介绍 webview/webview

[webview/webview](https://github.com/webview/webview) 是一个让你从 C/C++ 使用操作系统原生 WebView 组件（macOS 上的 WKWebView、Linux 上的 WebKitGTK、Windows 上的 WebView2）的库。与 Electron 不同，它不自带浏览器，对二进制大小的影响微乎其微。

我们用 CMake 拉取它。在 `CMakeLists.txt` 中添加以下内容：

```cmake
# webview/webview
FetchContent_Declare(webview
    GIT_REPOSITORY https://github.com/webview/webview
    GIT_TAG        master
)
FetchContent_MakeAvailable(webview)
```

这样就有可用的 `webview::core` CMake 目标了。用 `target_link_libraries` 链接它，就会自动设置好头文件路径和平台特定的框架。

> **macOS**：无需额外依赖。WKWebView 内置于系统。
>
> **Linux**：需要 WebKitGTK。用 `sudo apt install libwebkit2gtk-4.1-dev` 安装。
>
> **Windows**：需要 WebView2 运行时。Windows 11 已预装。Windows 10 请从 [微软官网](https://developer.microsoft.com/en-us/microsoft-edge/webview2/)下载。

## 6.2 在后台线程运行服务器

在第 5 章及之前，服务器的 `listen()` 会阻塞主线程。要使用 WebView，我们需要在另一个线程上运行服务器，而在主线程上运行 WebView 事件循环。

```cpp
#include "webview/webview.h"
#include <thread>

int main() {
  // ... (server setup is the same as Chapter 5) ...

  // Start the server on a background thread
  auto port = svr.bind_to_any_port("127.0.0.1");
  std::thread server_thread([&]() { svr.listen_after_bind(); });

  std::cout << "Listening on http://127.0.0.1:" << port << std::endl;

  // Display the UI with WebView
  webview::webview w(false, nullptr);
  w.set_title("Translate App");
  w.set_size(1024, 768, WEBVIEW_HINT_NONE);
  w.navigate("http://127.0.0.1:" + std::to_string(port));
  w.run(); // Block until the window is closed

  // Stop the server when the window is closed
  svr.stop();
  server_thread.join();
}
```

来看下几个关键点：

- **`bind_to_any_port`** — 不是 `listen("127.0.0.1", 8080)`，而是让操作系统选择一个可用端口。桌面应用可能被多次启动，用固定端口会冲突
- **`listen_after_bind`** — 在 `bind_to_any_port` 预留的端口上开始接受请求。`listen()` 一次性完成 bind 和 listen，但我们需要先知道端口号，所以把两个操作拆开
- **关闭顺序** — WebView 窗口关闭后，我们用 `svr.stop()` 停止服务器，并用 `server_thread.join()` 等待线程结束。如果顺序反过来，WebView 会失去对服务器的访问

第 5 章的 `signal_handler` 不再需要了。在桌面应用中，关闭窗口就意味着退出应用。

## 6.3 用 cpp-embedlib 嵌入静态文件

在第 5 章，我们从 `public/` 目录提供文件，所以需要把 `public/` 和二进制一起分发。用 [cpp-embedlib](https://github.com/yhirose/cpp-embedlib)，你可以把 HTML、CSS 和 JavaScript 嵌入二进制，把分发产物打包成单文件。

### CMakeLists.txt

拉取 cpp-embedlib 并嵌入 `public/`：

```cmake
# cpp-embedlib
FetchContent_Declare(cpp-embedlib
    GIT_REPOSITORY https://github.com/yhirose/cpp-embedlib
    GIT_TAG        main
)
FetchContent_MakeAvailable(cpp-embedlib)

# Embed the public/ directory into the binary
cpp_embedlib_add(WebAssets
    FOLDER    ${CMAKE_CURRENT_SOURCE_DIR}/public
    NAMESPACE Web
)

target_link_libraries(translate-app PRIVATE
    WebAssets                # Embedded files
    cpp-embedlib-httplib     # cpp-httplib integration
)
```

`cpp_embedlib_add` 在编译时把 `public/` 下的文件转换为二进制数据，并创建一个名为 `WebAssets` 的静态库。链接后，你就能通过 `Web::FS` 对象访问嵌入的文件。`cpp-embedlib-httplib` 是一个提供 `httplib::mount()` 函数的辅助库。

### 用 httplib::mount 替换 set_mount_point

只需把第 5 章的 `set_mount_point` 替换为 cpp-embedlib 的 `httplib::mount`：

```cpp
#include <cpp-embedlib-httplib.h>
#include "WebAssets.h"

// Chapter 5:
// svr.set_mount_point("/", "./public");

// Chapter 6:
httplib::mount(svr, Web::FS);
```

`httplib::mount` 会注册处理器，把 `Web::FS` 中嵌入的文件通过 HTTP 提供服务。MIME 类型会根据文件扩展名自动确定，无需手动设置 `Content-Type`。

文件内容直接映射到二进制的数据段，不会发生内存拷贝，也不会堆分配。

## 6.4 macOS：添加 Edit 菜单

如果你试着用 `Cmd+V` 向输入框粘贴文本，会发现不起作用。在 macOS 上，`Cmd+V`（粘贴）、`Cmd+C`（复制）等键盘快捷键经由应用的菜单栏路由。由于 webview/webview 不会创建菜单栏，这些快捷键永远传不到 WebView。我们需要用 Objective-C 运行时添加一个 macOS 的 Edit 菜单：

```cpp
#ifdef __APPLE__
#include <objc/objc-runtime.h>

void setup_macos_edit_menu() {
  auto cls    = [](const char *n) { return (id)objc_getClass(n); };
  auto sel    = sel_registerName;
  auto msg    = reinterpret_cast<id (*)(id, SEL)>(objc_msgSend);
  auto msg_s  = reinterpret_cast<id (*)(id, SEL, const char *)>(objc_msgSend);
  auto msg_id = reinterpret_cast<id (*)(id, SEL, id)>(objc_msgSend);
  auto msg_v  = reinterpret_cast<void (*)(id, SEL, id)>(objc_msgSend);
  auto msg_mi = reinterpret_cast<id (*)(id, SEL, id, SEL, id)>(objc_msgSend);

  auto str = [&](const char *s) {
    return msg_s(cls("NSString"), sel("stringWithUTF8String:"), s);
  };

  id app      = msg(cls("NSApplication"), sel("sharedApplication"));
  id mainMenu = msg(msg(cls("NSMenu"), sel("alloc")), sel("init"));
  id editItem = msg(msg(cls("NSMenuItem"), sel("alloc")), sel("init"));
  id editMenu = msg_id(msg(cls("NSMenu"), sel("alloc")),
                       sel("initWithTitle:"), str("Edit"));

  struct { const char *title; const char *action; const char *key; } items[] = {
    {"Undo",       "undo:",      "z"},
    {"Redo",       "redo:",      "Z"},
    {"Cut",        "cut:",       "x"},
    {"Copy",       "copy:",      "c"},
    {"Paste",      "paste:",     "v"},
    {"Select All", "selectAll:", "a"},
  };

  for (auto &[title, action, key] : items) {
    id mi = msg_mi(msg(cls("NSMenuItem"), sel("alloc")),
                   sel("initWithTitle:action:keyEquivalent:"),
                   str(title), sel(action), str(key));
    msg_v(editMenu, sel("addItem:"), mi);
  }

  msg_v(editItem, sel("setSubmenu:"), editMenu);
  msg_v(mainMenu, sel("addItem:"), editItem);
  msg_v(app, sel("setMainMenu:"), mainMenu);
}
#endif
```

在 `w.run()` 之前调用它：

```cpp
#ifdef __APPLE__
  setup_macos_edit_menu();
#endif
  w.run();
```

在 Windows 和 Linux 上，键盘快捷键不经菜单栏、直接送达当前控件，所以这个变通处理只需用在 macOS 上。

## 6.5 完整代码

<details>
<summary data-file="CMakeLists.txt">完整代码（CMakeLists.txt）</summary>

```cmake
cmake_minimum_required(VERSION 3.20)
project(translate-app CXX)
set(CMAKE_CXX_STANDARD 20)

include(FetchContent)

# llama.cpp
FetchContent_Declare(llama
    GIT_REPOSITORY https://github.com/ggml-org/llama.cpp
    GIT_TAG        master
    GIT_SHALLOW    TRUE
)
FetchContent_MakeAvailable(llama)

# cpp-httplib
FetchContent_Declare(httplib
    GIT_REPOSITORY https://github.com/yhirose/cpp-httplib
    GIT_TAG        master
)
FetchContent_MakeAvailable(httplib)

# nlohmann/json
FetchContent_Declare(json
    URL https://github.com/nlohmann/json/releases/download/v3.11.3/json.tar.xz
)
FetchContent_MakeAvailable(json)

# cpp-llamalib
FetchContent_Declare(cpp_llamalib
    GIT_REPOSITORY https://github.com/yhirose/cpp-llamalib
    GIT_TAG        main
)
FetchContent_MakeAvailable(cpp_llamalib)

# webview/webview
FetchContent_Declare(webview
    GIT_REPOSITORY https://github.com/webview/webview
    GIT_TAG        master
)
FetchContent_MakeAvailable(webview)

# cpp-embedlib
FetchContent_Declare(cpp-embedlib
    GIT_REPOSITORY https://github.com/yhirose/cpp-embedlib
    GIT_TAG        main
)
FetchContent_MakeAvailable(cpp-embedlib)

# Embed the public/ directory into the binary
cpp_embedlib_add(WebAssets
    FOLDER    ${CMAKE_CURRENT_SOURCE_DIR}/public
    NAMESPACE Web
)

find_package(OpenSSL REQUIRED)

add_executable(translate-app src/main.cpp)

target_link_libraries(translate-app PRIVATE
    httplib::httplib
    nlohmann_json::nlohmann_json
    cpp-llamalib
    OpenSSL::SSL OpenSSL::Crypto
    WebAssets
    cpp-embedlib-httplib
    webview::core
)

if(APPLE)
    target_link_libraries(translate-app PRIVATE
        "-framework CoreFoundation"
        "-framework Security"
    )
endif()

target_compile_definitions(translate-app PRIVATE
    CPPHTTPLIB_OPENSSL_SUPPORT
)
```

</details>

<details>
<summary data-file="main.cpp">完整代码（main.cpp）</summary>

```cpp
#include <httplib.h>
#include <nlohmann/json.hpp>
#include <cpp-llamalib.h>
#include <cpp-embedlib-httplib.h>
#include "WebAssets.h"
#include "webview/webview.h"

#ifdef __APPLE__
#include <objc/objc-runtime.h>
#endif

#include <algorithm>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <mutex>
#include <thread>

using json = nlohmann::json;

// -------------------------------------------------------------------------
// macOS Edit menu (Cmd+C/V/X/A require an Edit menu on macOS)
// -------------------------------------------------------------------------

#ifdef __APPLE__
void setup_macos_edit_menu() {
  auto cls    = [](const char *n) { return (id)objc_getClass(n); };
  auto sel    = sel_registerName;
  auto msg    = reinterpret_cast<id (*)(id, SEL)>(objc_msgSend);
  auto msg_s  = reinterpret_cast<id (*)(id, SEL, const char *)>(objc_msgSend);
  auto msg_id = reinterpret_cast<id (*)(id, SEL, id)>(objc_msgSend);
  auto msg_v  = reinterpret_cast<void (*)(id, SEL, id)>(objc_msgSend);
  auto msg_mi = reinterpret_cast<id (*)(id, SEL, id, SEL, id)>(objc_msgSend);

  auto str = [&](const char *s) {
    return msg_s(cls("NSString"), sel("stringWithUTF8String:"), s);
  };

  id app      = msg(cls("NSApplication"), sel("sharedApplication"));
  id mainMenu = msg(msg(cls("NSMenu"), sel("alloc")), sel("init"));
  id editItem = msg(msg(cls("NSMenuItem"), sel("alloc")), sel("init"));
  id editMenu = msg_id(msg(cls("NSMenu"), sel("alloc")),
                       sel("initWithTitle:"), str("Edit"));

  struct { const char *title; const char *action; const char *key; } items[] = {
    {"Undo",       "undo:",      "z"},
    {"Redo",       "redo:",      "Z"},
    {"Cut",        "cut:",       "x"},
    {"Copy",       "copy:",      "c"},
    {"Paste",      "paste:",     "v"},
    {"Select All", "selectAll:", "a"},
  };

  for (auto &[title, action, key] : items) {
    id mi = msg_mi(msg(cls("NSMenuItem"), sel("alloc")),
                   sel("initWithTitle:action:keyEquivalent:"),
                   str(title), sel(action), str(key));
    msg_v(editMenu, sel("addItem:"), mi);
  }

  msg_v(editItem, sel("setSubmenu:"), editMenu);
  msg_v(mainMenu, sel("addItem:"), editItem);
  msg_v(app, sel("setMainMenu:"), mainMenu);
}
#endif

// -------------------------------------------------------------------------
// Model definitions
// -------------------------------------------------------------------------

struct ModelInfo {
  std::string name;
  std::string params;
  std::string size;
  std::string repo;
  std::string filename;
};

const std::vector<ModelInfo> MODELS = {
  {
    .name     = "gemma-2-2b-it",
    .params   = "2B",
    .size     = "1.6 GB",
    .repo     = "bartowski/gemma-2-2b-it-GGUF",
    .filename = "gemma-2-2b-it-Q4_K_M.gguf",
  },
  {
    .name     = "gemma-2-9b-it",
    .params   = "9B",
    .size     = "5.8 GB",
    .repo     = "bartowski/gemma-2-9b-it-GGUF",
    .filename = "gemma-2-9b-it-Q4_K_M.gguf",
  },
  {
    .name     = "Llama-3.1-8B-Instruct",
    .params   = "8B",
    .size     = "4.9 GB",
    .repo     = "bartowski/Meta-Llama-3.1-8B-Instruct-GGUF",
    .filename = "Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf",
  },
};

// -------------------------------------------------------------------------
// Model storage directory
// -------------------------------------------------------------------------

std::filesystem::path get_models_dir() {
#ifdef _WIN32
  auto env = std::getenv("APPDATA");
  auto base = env ? std::filesystem::path(env) : std::filesystem::path(".");
  return base / "translate-app" / "models";
#else
  auto env = std::getenv("HOME");
  auto base = env ? std::filesystem::path(env) : std::filesystem::path(".");
  return base / ".translate-app" / "models";
#endif
}

// -------------------------------------------------------------------------
// Model download
// -------------------------------------------------------------------------

// Abort the download if progress_cb returns false
bool download_model(const ModelInfo &model,
                    std::function<bool(int)> progress_cb) {
  httplib::Client cli("https://huggingface.co");
  cli.set_follow_location(true);  // Hugging Face redirects to a CDN
  cli.set_read_timeout(std::chrono::hours(1)); // Long timeout for large models

  auto url = "/" + model.repo + "/resolve/main/" + model.filename;
  auto path = get_models_dir() / model.filename;
  auto tmp_path = std::filesystem::path(path).concat(".tmp");

  std::ofstream ofs(tmp_path, std::ios::binary);
  if (!ofs) { return false; }

  auto res = cli.Get(url,
    // content_receiver: Receive data chunk by chunk and write to file
    [&](const char *data, size_t len) {
      ofs.write(data, len);
      return ofs.good();
    },
    // progress: Report download progress (return false to abort)
    [&, last_pct = -1](size_t current, size_t total) mutable {
      int pct = total ? (int)(current * 100 / total) : 0;
      if (pct == last_pct) return true; // Skip if the value hasn't changed
      last_pct = pct;
      return progress_cb(pct);
    });

  ofs.close();

  if (!res || res->status != 200) {
    std::filesystem::remove(tmp_path);
    return false;
  }

  // Rename after download completes
  std::filesystem::rename(tmp_path, path);
  return true;
}

// -------------------------------------------------------------------------
// Server
// -------------------------------------------------------------------------

int main() {
  httplib::Server svr;
  // Create the model storage directory
  auto models_dir = get_models_dir();
  std::filesystem::create_directories(models_dir);

  // Auto-download the default model if not already present
  std::string selected_model = MODELS[0].filename;
  auto path = models_dir / selected_model;
  if (!std::filesystem::exists(path)) {
    std::cout << "Downloading " << selected_model << "..." << std::endl;
    if (!download_model(MODELS[0], [](int pct) {
          std::cout << "\r" << pct << "%" << std::flush;
          return true;
        })) {
      std::cerr << "\nFailed to download model." << std::endl;
      return 1;
    }
    std::cout << std::endl;
  }
  auto llm = llamalib::Llama{path};
  std::mutex llm_mutex; // Protect access during model switching

  // Set a long timeout since LLM inference takes time (default is 5 seconds)
  svr.set_read_timeout(300);
  svr.set_write_timeout(300);

  svr.set_logger([](const auto &req, const auto &res) {
    std::cout << req.method << " " << req.path << " -> " << res.status
              << std::endl;
  });

  svr.Get("/health", [](const httplib::Request &, httplib::Response &res) {
    res.set_content(json{{"status", "ok"}}.dump(), "application/json");
  });

  // --- Translation endpoint (Chapter 2) ------------------------------------

  svr.Post("/translate",
           [&](const httplib::Request &req, httplib::Response &res) {
    auto input = json::parse(req.body, nullptr, false);
    if (input.is_discarded()) {
      res.status = 400;
      res.set_content(json{{"error", "Invalid JSON"}}.dump(),
                      "application/json");
      return;
    }

    if (!input.contains("text") || !input["text"].is_string() ||
        input["text"].get<std::string>().empty()) {
      res.status = 400;
      res.set_content(json{{"error", "'text' is required"}}.dump(),
                      "application/json");
      return;
    }

    auto text = input["text"].get<std::string>();
    auto target_lang = input.value("target_lang", "ja");

    auto prompt = "Translate the following text to " + target_lang +
                  ". Output only the translation, nothing else.\n\n" + text;

    try {
      std::lock_guard<std::mutex> lock(llm_mutex);
      auto translation = llm.chat(prompt);
      res.set_content(json{{"translation", translation}}.dump(),
                      "application/json");
    } catch (const std::exception &e) {
      res.status = 500;
      res.set_content(json{{"error", e.what()}}.dump(), "application/json");
    }
  });

  // --- SSE streaming translation (Chapter 3) -------------------------------

  svr.Post("/translate/stream",
           [&](const httplib::Request &req, httplib::Response &res) {
    auto input = json::parse(req.body, nullptr, false);
    if (input.is_discarded()) {
      res.status = 400;
      res.set_content(json{{"error", "Invalid JSON"}}.dump(),
                      "application/json");
      return;
    }

    if (!input.contains("text") || !input["text"].is_string() ||
        input["text"].get<std::string>().empty()) {
      res.status = 400;
      res.set_content(json{{"error", "'text' is required"}}.dump(),
                      "application/json");
      return;
    }

    auto text = input["text"].get<std::string>();
    auto target_lang = input.value("target_lang", "ja");

    auto prompt = "Translate the following text to " + target_lang +
                  ". Output only the translation, nothing else.\n\n" + text;

    res.set_chunked_content_provider(
        "text/event-stream",
        [&, prompt](size_t, httplib::DataSink &sink) {
          std::lock_guard<std::mutex> lock(llm_mutex);
          try {
            llm.chat(prompt, [&](std::string_view token) {
              sink.os << "data: "
                      << json(std::string(token)).dump(
                           -1, ' ', false, json::error_handler_t::replace)
                      << "\n\n";
              return sink.os.good(); // Abort inference on disconnect
            });
            sink.os << "data: [DONE]\n\n";
          } catch (const std::exception &e) {
            sink.os << "data: " << json({{"error", e.what()}}).dump() << "\n\n";
          }
          sink.done();
          return true;
        });
  });

  // --- Model list (Chapter 4) ----------------------------------------------

  svr.Get("/models",
          [&](const httplib::Request &, httplib::Response &res) {
    auto models_dir = get_models_dir();
    auto arr = json::array();
    for (const auto &m : MODELS) {
      auto path = models_dir / m.filename;
      arr.push_back({
        {"name",       m.name},
        {"params",     m.params},
        {"size",       m.size},
        {"downloaded", std::filesystem::exists(path)},
        {"selected",   m.filename == selected_model},
      });
    }
    res.set_content(json{{"models", arr}}.dump(), "application/json");
  });

  // --- Model selection (Chapter 4) -----------------------------------------

  svr.Post("/models/select",
           [&](const httplib::Request &req, httplib::Response &res) {
    auto input = json::parse(req.body, nullptr, false);
    if (input.is_discarded() || !input.contains("model")) {
      res.status = 400;
      res.set_content(json{{"error", "'model' is required"}}.dump(),
                      "application/json");
      return;
    }

    auto name = input["model"].get<std::string>();

    auto it = std::find_if(MODELS.begin(), MODELS.end(),
      [&](const ModelInfo &m) { return m.name == name; });

    if (it == MODELS.end()) {
      res.status = 404;
      res.set_content(json{{"error", "Unknown model"}}.dump(),
                      "application/json");
      return;
    }

    const auto &model = *it;

    // Always respond with SSE (same format whether downloaded or not)
    res.set_chunked_content_provider(
        "text/event-stream",
        [&, model](size_t, httplib::DataSink &sink) {
          // SSE event sending helper
          auto send = [&](const json &event) {
            sink.os << "data: " << event.dump() << "\n\n";
          };

          // Download if not yet downloaded (report progress via SSE)
          auto path = get_models_dir() / model.filename;
          if (!std::filesystem::exists(path)) {
            bool ok = download_model(model, [&](int pct) {
              send({{"status", "downloading"}, {"progress", pct}});
              return sink.os.good(); // Abort download on client disconnect
            });
            if (!ok) {
              send({{"status", "error"}, {"message", "Download failed"}});
              sink.done();
              return true;
            }
          }

          // Load and switch to the model
          send({{"status", "loading"}});
          {
            std::lock_guard<std::mutex> lock(llm_mutex);
            llm = llamalib::Llama{path};
            selected_model = model.filename;
          }

          send({{"status", "ready"}});
          sink.done();
          return true;
        });
  });

  // --- Embedded file serving (Chapter 6) ------------------------------------
  // Chapter 5: svr.set_mount_point("/", "./public");
  httplib::mount(svr, Web::FS);

  // Start the server on a background thread
  auto port = svr.bind_to_any_port("127.0.0.1");
  std::thread server_thread([&]() { svr.listen_after_bind(); });

  std::cout << "Listening on http://127.0.0.1:" << port << std::endl;

  // Display the UI with WebView
  webview::webview w(false, nullptr);
  w.set_title("Translate App");
  w.set_size(1024, 768, WEBVIEW_HINT_NONE);
  w.navigate("http://127.0.0.1:" + std::to_string(port));

#ifdef __APPLE__
  setup_macos_edit_menu();
#endif
  w.run(); // Block until the window is closed

  // Stop the server when the window is closed
  svr.stop();
  server_thread.join();
}
```

</details>

总结与第 5 章的变化：

- `#include <csignal>` 换成了 `#include <thread>`、`<cpp-embedlib-httplib.h>`、`"WebAssets.h"`、`"webview/webview.h"`
- 删掉了 `signal_handler` 函数
- `svr.set_mount_point("/", "./public")` 换成了 `httplib::mount(svr, Web::FS)`
- `svr.listen("127.0.0.1", 8080)` 换成了 `bind_to_any_port` + `listen_after_bind` + WebView 事件循环

处理器的代码一行都没改。到第 5 章为止构建的 REST API、SSE 流式输出和模型管理全部原样可用。

## 6.6 构建与测试

```bash
cmake -B build
cmake --build build -j
```

启动应用：

```bash
./build/translate-app
```

不需要浏览器。窗口会自动打开。第 5 章的那个 UI 原样出现，翻译和模型切换也都照常工作。

关闭窗口时，服务器会自动关闭。无需 `Ctrl+C`。

### 需要分发什么

你只需分发：

- 单个 `translate-app` 二进制

仅此而已。不需要 `public/` 目录。HTML、CSS 和 JavaScript 已嵌入二进制。模型文件在首次启动时自动下载，所以无需让用户提前准备任何东西。

## 下一章

恭喜你！🎉

第 1 章里，`/health` 只会返回 `{"status":"ok"}`。现在我们有了一个桌面应用：输入文本就能实时流式返回翻译，从下拉框选一个不同的模型就会自动下载，关闭窗口就能干净地关闭一切——所有这些都在一个可分发的单二进制里。

本章我们改变的只是静态文件服务和服务器启动方式。处理器的代码一行都没改。到第 5 章为止构建的 REST API、SSE 流式输出和模型管理，作为桌面应用原样可用。

下一章，我们切换视角，通读 llama.cpp 自带的 `llama-server` 代码。把我们的简单服务器与生产级服务器对比，看看设计决策有何不同、又为何不同。

**下一章：**[阅读 llama.cpp 服务器源码](../ch07-code-reading)
