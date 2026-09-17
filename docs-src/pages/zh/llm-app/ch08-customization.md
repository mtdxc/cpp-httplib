---
title: "8. 把它变成你自己的"
order: 8

---

到第 7 章为止，我们已经构建了一个翻译桌面应用，并研究了生产级代码的差异。本章我们来梳理一下**把这个应用彻底变成你自己作品**的关键要点。

翻译应用只是一个载体。把 llama.cpp 换成你自己的库，同样的架构适用于任何应用。

## 8.1 替换构建配置

首先，把 `CMakeLists.txt` 中与 llama.cpp 相关的 `FetchContent` 条目替换为你自己的库。

```cmake
# Remove: llama.cpp and cpp-llamalib FetchContent

# Add: your own library
FetchContent_Declare(my_lib
    GIT_REPOSITORY https://github.com/yourname/my-lib
    GIT_TAG        main
)
FetchContent_MakeAvailable(my_lib)

target_link_libraries(my-app PRIVATE
    httplib::httplib
    nlohmann_json::nlohmann_json
    my_lib        # Your library instead of cpp-llamalib
    # ...
)
```

如果你的库不支持 CMake，可以直接把头文件和源文件放进 `src/`，并把它们加到 `add_executable`。cpp-httplib、nlohmann/json 和 webview 保持不变即可。

## 8.2 让 API 适配你的任务

根据你的任务修改翻译 API 的端点和参数。

| 翻译应用 | 你的应用（例：图像处理） |
|---|---|
| `POST /translate` | `POST /process` |
| `{"text": "...", "target_lang": "ja"}` | `{"image": "base64...", "filter": "blur"}` |
| `POST /translate/stream` | `POST /process/stream` |
| `GET /models` | `GET /filters` 或 `GET /presets` |

然后更新每个处理器的实现。例如，只需把 `llm.chat()` 调用替换为你自己库的 API。

```cpp
// Before: LLM translation
auto translation = llm.chat(prompt);
res.set_content(json{{"translation", translation}}.dump(), "application/json");

// After: e.g., an image processing library
auto result = my_lib::process(input_image, options);
res.set_content(json{{"result", result}}.dump(), "application/json");
```

SSE 也是如此。如果你的库有通过回调报告进度的函数，你可以用与第 3 章完全相同的模式发送增量响应。SSE 并不限于 LLM——它对任何耗时任务都很有用：图像处理进度、数据转换步骤、长时间计算。

## 8.3 设计上的考虑

### 初始化代价高的库

在本书中，我们在 `main()` 开头加载 LLM 模型并保存在变量里。这是有意为之的。如果每次请求都重新加载模型会花上好几秒，所以我们在启动时加载一次并复用。如果你的库初始化代价高（加载大数据文件、获取 GPU 资源等），同样的做法也很适用。

### 线程安全

cpp-httplib 用线程池并发处理请求。在第 4 章中，我们用 `std::mutex` 保护 `llm` 对象，以防止切换模型时崩溃。集成你自己的库时同样适用这个模式。如果你的库不是线程安全的，或者你需要在运行时替换对象，就用 `std::mutex` 保护访问。

## 8.4 自定义 UI

编辑 `public/` 中的三个文件。

- **`index.html`** — 修改输入表单布局。把 `<textarea>` 换成 `<input type="file">`、添加参数字段等
- **`style.css`** — 调整布局和颜色。保留双栏设计或改为单栏
- **`script.js`** — 更新 `fetch()` 的目标 URL、请求体和响应展示方式

即使不改任何服务器代码，只换掉 HTML 也能让应用看起来完全不同。因为这些是静态文件，你可以快速迭代——无需重启服务器，只需刷新浏览器。

本书使用了纯 HTML、CSS 和 JavaScript，但配合 Vue 或 React 等前端框架，或一个 CSS 框架，你能构建出更精致的应用。

## 8.5 分发上的考虑

### 许可证

检查你所用库的许可证。cpp-httplib（MIT）、nlohmann/json（MIT）和 webview（MIT）都允许商用。别忘了也检查你自己的库及其依赖的许可证。

### 模型与数据文件

我们在第 4 章构建的下载机制不限于 LLM 模型。如果你的应用需要大数据文件，同样的模式也可以让它在首次启动时自动下载，既保持二进制小巧，又免去用户手动配置。

如果数据很小，可以用 cpp-embedlib 直接把它嵌入二进制文件。

### 跨平台构建

webview 支持 macOS、Linux 和 Windows。为各平台构建时：

- **macOS** — 无额外依赖
- **Linux** — 需要 `libwebkit2gtk-4.1-dev`
- **Windows** — 需要 WebView2 运行时（Windows 11 预装）

也可以考虑在 CI（如 GitHub Actions）中配置跨平台构建。

## 结语

非常感谢你读到最后。🙏

本书从第 1 章 `/health` 返回 `{"status":"ok"}` 开始。从那以后我们构建了 REST API、添加了 SSE 流式输出、从 Hugging Face 下载模型、创建了基于浏览器的 Web UI，最后把它们打包成单二进制的桌面应用。第 7 章我们通读了 `llama-server` 的代码，了解了生产级服务器在设计上的不同。这一路走了不少，真心感谢你一路坚持到最后。

回顾起来，我们亲手用到了好几种 cpp-httplib 的关键功能：

- **服务器**：路由、JSON 响应、用 `set_chunked_content_provider` 做 SSE 流式输出、用 `set_mount_point` 提供静态文件
- **客户端**：HTTPS 连接、重定向跟随、用 content receiver 大文件下载、进度回调
- **WebView 集成**：用 `bind_to_any_port` + `listen_after_bind` 做后台线程

cpp-httplib 还有许多本节未涉及的功能，包括 multipart 文件上传、认证、超时控制、压缩和范围请求。详见 [cpp-httplib 向导](../../tour/)。

这些模式并不限于翻译应用。如果你想给自己的 C++ 库加上 Web API、给它一个浏览器 UI，或把它打包成易于分发的桌面应用—我希望本书能成为一份有用的参考。

拿上你自己的库，构建你自己的应用，玩得开心。Happy hacking！🚀
