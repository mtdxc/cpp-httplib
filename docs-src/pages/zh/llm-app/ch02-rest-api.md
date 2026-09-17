---
title: "2. 集成 llama.cpp 并构建 REST API"
order: 2

---

在第 1 章的骨架中，`/translate` 只是返回 `"TODO"`。本章我们集成 llama.cpp 推理，把它变成真正能返回翻译结果的 API。

直接调用 llama.cpp 的 API 会让代码相当长，所以我们使用一个名为 [cpp-llamalib](https://github.com/yhirose/cpp-llamalib) 的轻量封装库。它让你只需几行代码就能加载模型并运行推理，从而把重点放在 cpp-httplib 上。

## 2.1 初始化 LLM

只需把模型文件路径传给 `llamalib::Llama`，模型加载、上下文创建和采样器配置，都全帮你处理好了。如果你在第 1 章下载了不同的模型，请相应调整路径。

```cpp
#include <cpp-llamalib.h>

int main() {
  auto llm = llamalib::Llama{"models/gemma-2-2b-it-Q4_K_M.gguf"};

  // LLM inference takes time, so set a longer timeout (default is 5 seconds)
  svr.set_read_timeout(300);
  svr.set_write_timeout(300);

  // ... Build and start the HTTP server ...
}
```

如果你想修改 GPU 层数、上下文长度或其他设置，可以通过 `llamalib::Options` 指定。

```cpp
auto llm = llamalib::Llama{"models/gemma-2-2b-it-Q4_K_M.gguf", {
  .n_gpu_layers = 0,  // CPU only
  .n_ctx = 4096,
}};
```

## 2.2 `/translate` 处理器

我们把第 1 章中返回占位 JSON 的处理器替换为真正的推理。

```cpp
svr.Post("/translate",
         [&](const httplib::Request &req, httplib::Response &res) {
  // Parse JSON (3rd arg `false`: don't throw on failure, check with `is_discarded()`)
  auto input = json::parse(req.body, nullptr, false);
  if (input.is_discarded()) {
    res.status = 400;
    res.set_content(json{{"error", "Invalid JSON"}}.dump(),
                    "application/json");
    return;
  }

  // Validate required fields
  if (!input.contains("text") || !input["text"].is_string() ||
      input["text"].get<std::string>().empty()) {
    res.status = 400;
    res.set_content(json{{"error", "'text' is required"}}.dump(),
                    "application/json");
    return;
  }

  auto text = input["text"].get<std::string>();
  auto target_lang = input.value("target_lang", "ja"); // Default is Japanese

  // Build the prompt and run inference
  auto prompt = "Translate the following text to " + target_lang +
                ". Output only the translation, nothing else.\n\n" + text;

  try {
    auto translation = llm.chat(prompt);
    res.set_content(json{{"translation", translation}}.dump(),
                    "application/json");
  } catch (const std::exception &e) {
    res.status = 500;
    res.set_content(json{{"error", e.what()}}.dump(), "application/json");
  }
});
```

`llm.chat()` 在推理过程中可能抛出异常（如上下文长度过长时）。用 `try/catch` 捕获它们并以 JSON 返回错误，以防止服务器崩溃。

## 2.3 完整代码

以下是当前所有修改的最终代码。

<details>
<summary data-file="main.cpp">完整代码（main.cpp）</summary>

```cpp
#include <httplib.h>
#include <nlohmann/json.hpp>
#include <cpp-llamalib.h>

#include <csignal>
#include <iostream>

using json = nlohmann::json;

httplib::Server svr;

// Graceful shutdown on `Ctrl+C`
void signal_handler(int sig) {
  if (sig == SIGINT || sig == SIGTERM) {
    std::cout << "\nReceived signal, shutting down gracefully...\n";
    svr.stop();
  }
}

int main() {
  // Load the model downloaded in Chapter 1
  auto llm = llamalib::Llama{"models/gemma-2-2b-it-Q4_K_M.gguf"};

  // LLM inference takes time, so set a longer timeout (default is 5 seconds)
  svr.set_read_timeout(300);
  svr.set_write_timeout(300);

  // Log requests and responses
  svr.set_logger([](const auto &req, const auto &res) {
    std::cout << req.method << " " << req.path << " -> " << res.status
              << std::endl;
  });

  svr.Get("/health", [](const httplib::Request &, httplib::Response &res) {
    res.set_content(json{{"status", "ok"}}.dump(), "application/json");
  });

  svr.Post("/translate",
           [&](const httplib::Request &req, httplib::Response &res) {
    // Parse JSON (3rd arg `false`: don't throw on failure, check with `is_discarded()`)
    auto input = json::parse(req.body, nullptr, false);
    if (input.is_discarded()) {
      res.status = 400;
      res.set_content(json{{"error", "Invalid JSON"}}.dump(),
                      "application/json");
      return;
    }

    // Validate required fields
    if (!input.contains("text") || !input["text"].is_string() ||
        input["text"].get<std::string>().empty()) {
      res.status = 400;
      res.set_content(json{{"error", "'text' is required"}}.dump(),
                      "application/json");
      return;
    }

    auto text = input["text"].get<std::string>();
    auto target_lang = input.value("target_lang", "ja"); // Default is Japanese

    // Build the prompt and run inference
    auto prompt = "Translate the following text to " + target_lang +
                  ". Output only the translation, nothing else.\n\n" + text;

    try {
      auto translation = llm.chat(prompt);
      res.set_content(json{{"translation", translation}}.dump(),
                      "application/json");
    } catch (const std::exception &e) {
      res.status = 500;
      res.set_content(json{{"error", e.what()}}.dump(), "application/json");
    }
  });

  // Dummy implementations to be replaced with real ones in later chapters
  svr.Get("/models",
          [](const httplib::Request &, httplib::Response &res) {
    res.set_content(json{{"models", json::array()}}.dump(), "application/json");
  });

  svr.Post("/models/select",
           [](const httplib::Request &, httplib::Response &res) {
    res.set_content(json{{"status", "TODO"}}.dump(), "application/json");
  });

  // Allow the server to be stopped with `Ctrl+C` (`SIGINT`) or `kill` (`SIGTERM`)
  signal(SIGINT, signal_handler);
  signal(SIGTERM, signal_handler);

  // Start the server (blocks until `stop()` is called)
  std::cout << "Listening on http://127.0.0.1:8080" << std::endl;
  svr.listen("127.0.0.1", 8080);
}
```

</details>

## 2.4 试一试

重新构建并启动服务器，验证它现在能返回真正的翻译结果。

```bash
cmake --build build -j
./build/translate-server
```

```bash
curl -X POST http://localhost:8080/translate \
  -H "Content-Type: application/json" \
  -d '{"text": "I had a great time visiting Tokyo last spring. The cherry blossoms were beautiful.", "target_lang": "ja"}'
# => {"translation":"去年の春に東京を訪れた。桜が綺麗だった。"}
```

第 1 章里返回是 `"TODO"`，而现在你将得到真正的内容。

## 下一章

本章构建的 REST API 会等整个翻译完成后才发送响应，对长文本来说，用户只能等待，无法得到进展提示。

下一章，我们用 SSE（Server-Sent Events）在 token 生成的同时，实时流式返回。

**下一章：**[用 SSE 实现 token 流式输出](../ch03-sse-streaming)
