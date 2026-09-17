---
title: "用 cpp-httplib 构建桌面 LLM App"
order: 0

---

有没有想过给你自己的 C++ 库加上 Web API，或者快速构建一个类似 Electron 的桌面应用？在 Rust 里你可能会想到 "Tauri + axum"，但在 C++ 里这似乎一直遥不可及。

有了 [cpp-httplib](https://github.com/yhirose/cpp-httplib)、[webview/webview](https://github.com/webview/webview) 和 [cpp-embedlib](https://github.com/yhirose/cpp-embedlib)，你可以用 C++ 走同样的路线——并且产出一个小巧、易于分发的二进制文件。

本教程将使用 [llama.cpp](https://github.com/ggml-org/llama.cpp) 构建一个由 LLM 驱动的翻译应用，从 "REST API" 到 "SSE 流式输出"，再到 "Web UI"，最后到 "桌面应用"，一步一步推进。翻译只是一个载体——把 llama.cpp 换成你自己的库，同样的架构适用于任何应用。

![Desktop App](app.png#large-center)

只要你懂基本的 C++17，了解 HTTP / REST API 的基础知识，就可以开始了。

## 章节

1. **[搭建项目环境](ch01-setup)** — 获取依赖、配置构建、编写骨架代码
2. **[集成llama.cpp 并构建 REST API](ch02-rest-api)** — 以 JSON 返回翻译结果
3. **[用 SSE 实现逐 token 流式输出](ch03-sse-streaming)** — 逐 token 流式返回响应
4. **[添加模型下载与管理](ch04-model-management)** — 从 Hugging Face 下载并切换模型
5. **[添加 Web UI](ch05-web-ui)** — 基于浏览器的翻译界面
6. **[用 WebView 变成桌面应用](ch06-desktop-app)** — 单二进制桌面应用程序
7. **[阅读 llama.cpp 服务器源码](ch07-code-reading)** — 与生产级代码对照
8. **[把它变成你自己的](ch08-customization)** — 换成你自己的库并自由定制
