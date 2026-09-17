---
title: "3. 用 SSE 实现 token 流式输出"
order: 3

---

第 2 章的 `/translate` 端点是在翻译全部完成后一次性返回整个结果。对短句子来说这没问题，但对长文本而言，用户要盯着空白屏幕等上好几秒。

本章我们添加一个 `/translate/stream` 端点，用 SSE（Server-Sent Events）在 token 生成的同时实时返回。这正是 ChatGPT 和 Claude API 采用的方式。

## 3.1 什么是 SSE？

SSE 是一种把 HTTP 响应作为流来发送的方式。客户端发出请求后，服务器保持连接打开，逐渐返回事件。格式很简单，是纯文本。

```text
data: "去年の"
data: "春に"
data: "東京を"
data: [DONE]
```

每行以 `data:` 开头，事件之间用空行分隔。Content-Type 是 `text/event-stream`。token 以转义后的 JSON 字符串发送，所以外面会包着一对双引号（我们在 3.3 节实现这一点）。

## 3.2 用 cpp-httplib 实现流式输出

在 cpp-httplib 中，可以用 `set_chunked_content_provider` 增量发送响应。每次在回调里向 `sink.os` 写入，数据就会被发送给客户端。

```cpp
res.set_chunked_content_provider(
    "text/event-stream",
    [](size_t offset, httplib::DataSink &sink) {
      sink.os << "data: hello\n\n";
      sink.done();
      return true;
    });
```

调用 `sink.done()` 会结束流。如果客户端在流中途断开，向 `sink.os` 写入会失败，此时 `sink.os.fail()` 会返回 `true`。你可以利用这一点检测断开，并中止不必要的推理。

## 3.3 `/translate/stream` 处理器

JSON 解析和校验与第 2 章的 `/translate` 端点完全相同，唯一的区别是响应的返回方式。我们把 `llm.chat()` 的流式回调与 `set_chunked_content_provider` 结合起来。

```cpp
svr.Post("/translate/stream",
         [&](const httplib::Request &req, httplib::Response &res) {
  // ... JSON parsing and validation same as /translate ...

  res.set_chunked_content_provider(
      "text/event-stream",
      [&, prompt](size_t, httplib::DataSink &sink) {
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
```

几个关键点：

- 给 `llm.chat()` 传入回调后，每生成一个 token 就会调用一次。如果回调返回 `false`，生成会被中止
- 向 `sink.os` 写入后，可以用 `sink.os.good()` 检查客户端是否仍连接着。如果客户端已断开，就返回 `false` 停止推理
- 每个 token 发送前都用 `json(token).dump()` 转义为 JSON 字符串。即使 token 中含有换行或引号也是安全的
- `dump(-1, ' ', false, ...)` 的前三个参数是默认值，关键是第四个参数 `json::error_handler_t::replace`。由于 LLM 按 subword 级别返回 token，多字节字符（如日语）可能被拆到相邻的 token 中间。把不完整的 UTF-8 字节序列直接传给 `dump()` 会抛异常，用 `replace` 可以安全地替换它们。浏览器端会重新拼合字节，可以正常显示
- 整个 lambda 包在 `try/catch` 里。`llm.chat()` 可能因超出上下文窗口等原因抛异常。如果异常在 lambda 里未被捕获，服务器会崩溃，因此我们把错误作为 SSE 事件返回
- `data: [DONE]` 遵循 OpenAI API 的惯例，向客户端表示流已结束

## 3.4 完整代码

下面是在第 2 章代码基础上添加了 `/translate/stream` 端点的完整代码。

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
  // Load the GGUF model
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

  // Standard translation endpoint from Chapter 2
  svr.Post("/translate",
           [&](const httplib::Request &req, httplib::Response &res) {
    // JSON parsing and validation (see Chapter 2 for details)
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
      auto translation = llm.chat(prompt);
      res.set_content(json{{"translation", translation}}.dump(),
                      "application/json");
    } catch (const std::exception &e) {
      res.status = 500;
      res.set_content(json{{"error", e.what()}}.dump(), "application/json");
    }
  });

  // SSE streaming translation endpoint
  svr.Post("/translate/stream",
           [&](const httplib::Request &req, httplib::Response &res) {
    // JSON parsing and validation (same as /translate)
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

  // Dummy implementations to be replaced in later chapters
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

## 3.5 试一试

编译并启动服务器。

```bash
cmake --build build -j
./build/translate-server
```

使用 curl 的 `-N` 选项禁用缓冲，当token到达时，你就能实时看到 token 的显示。

```bash
curl -N -X POST http://localhost:8080/translate/stream \
  -H "Content-Type: application/json" \
  -d '{"text": "I had a great time visiting Tokyo last spring. The cherry blossoms were beautiful.", "target_lang": "ja"}'
```

```text
data: "去年の"
data: "春に"
data: "東京を"
data: "訪れた"
data: "。"
data: "桜が"
data: "綺麗だった"
data: "。"
data: [DONE]
```

你应该能看到 token 一个个流式输出。第 2 章的 `/translate` 端点也仍然正常工作。

## 下一章

服务器的翻译功能到此已经完整。下一章我们用 cpp-httplib 的客户端功能，添加从 Hugging Face 获取和管理模型的能力。

**下一章：**[添加模型下载和管理](../ch04-model-management)
