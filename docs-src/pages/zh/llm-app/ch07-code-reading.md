---
title: "7. 阅读 llama.cpp 服务器源码"
order: 7

---

前六章我们从零开始构建了一个翻译桌面应用。它已是一个可用的产品，但归根到底是「学习导向」的实现。那么「生产级」代码有什么不同？我们来读一读 llama.cpp 自带的官方服务器 `llama-server` 的源码，做个对比。

`llama-server` 位于 `llama.cpp/tools/server/`。它用的是同一个 cpp-httplib，所以你可以像前面章节那样阅读它的代码。

## 7.1 源码位置

```ascii
llama.cpp/tools/server/
├── server.cpp           # Main server implementation
├── httplib.h            # cpp-httplib (bundled version)
└── ...
```

代码都集中在单个 `server.cpp` 里。它有数千行之多，但只要理解了结构，你就能锁定值得一读的部分。

## 7.2 OpenAI 兼容的 API

我们构建的服务器与 `llama-server` 最大的区别在于 API 设计。

**我们的 API：**

```text
POST /translate          → {"translation": "..."}
POST /translate/stream   → SSE: data: "token"
```

**llama-server 的 API：**

```text
POST /v1/chat/completions  → OpenAI-compatible JSON
POST /v1/completions       → OpenAI-compatible JSON
POST /v1/embeddings        → Text embedding vectors
```

`llama-server` 遵循 [OpenAI 的 API 规范](https://platform.openai.com/docs/api-reference)。这意味着对 OpenAI 的官方客户端库（如 Python 的 `openai` 包）开箱即用。

```python
# Example of connecting to llama-server with the OpenAI client
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8080/v1", api_key="dummy")

response = client.chat.completions.create(
    model="local-model",
    messages=[{"role": "user", "content": "Hello!"}]
)
```

与现有工具和库的兼容性是一个重大的设计决策。我们设计的是一个简单的翻译专用 API，但如果你要做的是通用服务器，OpenAI 兼容已经成为准标准。

## 7.3 并发请求处理

我们的服务器一次只处理一个请求。如果翻译中又来另一个请求，它要等上一次推理结束才能处理。对单人使用的桌面应用来说这没问题，但对多用户共享的服务器就成了问题。

`llama-server` 通过一种叫作 **slots（槽位）** 的机制处理并发请求。

![llama-server 的 slot 管理](../slots.svg#half)

关键在于，各 slot 的 token 不是**逐个串行推理**，而是**作为单个批次一次性推理**。GPU 擅并行，同时处理两个用户和处理一个人花的时间几乎一样。这就是所谓的「连续批处理（continuous batching）」。

在我们的服务器中，cpp-httplib 的线程池为每个请求分配一个线程，但推理本身在 `llm.chat()` 内单线程运行。`llama-server` 把这个推理步骤合并进了一个共享的批处理循环。

## 7.4 SSE 格式的差异

流式机制本身是一样的（`set_chunked_content_provider` + SSE），但数据格式不同。

**我们的格式：**

```text
data: "去年の"
data: "春に"
data: [DONE]
```

**llama-server（OpenAI 兼容）：**

```text
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"content":"去年の"}}]}
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"content":"春に"}}]}
data: [DONE]
```

我们的格式只发送 token。而 `llama-server` 遵循 OpenAI 规范，即使一个 token 也要包在 JSON 里。看起来啰嗦，但它包含了客户端用得上的信息，比如用于标识请求的 `id`，以及说明生成为何停止的 `finish_reason`。

## 7.5 KV 缓存复用

在我们的服务器中，每次请求都从头处理整个提示词。我们翻译应用的提示词很短（"Translate the following text to ja..." + 输入文本），所以不是问题。

`llama-server` 在请求与之前的请求共享相同提示词前缀时，会复用前缀部分的 KV 缓存。

![KV 缓存复用](../kv-cache.svg#half)

对于每次请求都发送长系统提示词和 few-shot 示例的聊天机器人，仅这一项就能大幅缩短响应时间。这是天壤之别：每次都处理几千 token 的系统提示词，还是瞬间从缓存读取。

对我们的翻译应用来说，系统提示词只有一句话，收益有限。但当你把它应用到自己的项目时，这是一个值得记住的优化。

## 7.6 结构化输出

由于我们的翻译 API 返回纯文本，没必要约束输出格式。但如果你希望 LLM 以 JSON 应答呢？

```text
Prompt: Analyze the sentiment of the following text and return it as JSON.
LLM output (expected): {"sentiment": "positive", "score": 0.8}
LLM output (reality): Here are the results of the sentiment analysis. {"sentiment": ...
```

LLM 有时会无视指令、加上多余的文字。`llama-server` 用**语法约束（grammar constraints）**解决了这个问题。

```bash
curl http://localhost:8080/v1/chat/completions \
  -d '{
    "messages": [{"role": "user", "content": "Analyze sentiment..."}],
    "json_schema": {
      "type": "object",
      "properties": {
        "sentiment": {"type": "string", "enum": ["positive", "negative", "neutral"]},
        "score": {"type": "number"}
      },
      "required": ["sentiment", "score"]
    }
  }'
```

指定 `json_schema` 后，生成 token 时不符合语法的 token 会被排除。这保证输出永远是合法 JSON，不必担心 `json::parse` 失败。

把 LLM 嵌入应用时，能否可靠地解析输出直接影响可靠性。翻译这类自由文本输出不需要语法约束，但对于需要把结构化数据作为 API 响应返回的场景，它是不可或缺的。

## 7.7 小结

把我们讲过的差异整理如下。

| 方面 | 我们的服务器 | llama-server |
|------|-------------|--------------|
| API 设计 | 翻译专用 | OpenAI 兼容 |
| 并发请求 | 串行处理 | Slots + 连续批处理 |
| SSE 格式 | 只有token | OpenAI 兼容 JSON |
| KV 缓存 | 每次清空 | 前缀复用 |
| 结构化输出 | 无 | JSON Schema / 语法约束 |
| 代码规模 | 约 200 行 | 数千行 |

我们的代码之所以简洁，是因为前提就是「作为桌面应用由单人使用」。如果你要为多个用户构建服务器，或者要融入现有生态，`llama-server` 的设计就是非常有价值的参考。

反过来说，200 行代码也足以做出功能完整的翻译应用。希望这次源码阅读也能让你体会到「只造需要的东西」的价值。

## 下一章

下一章，我们来梳理换成你自己的库、自定义应用，把它真正变成你自己作品的关键要点。

**下一章：**[把它变成你自己的](../ch08-customization)
