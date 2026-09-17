---
title: "5. 添加 Web UI"
order: 5

---

到第 4 章结束时，我们已经实现了所有服务器功能：翻译 API、SSE 流式输出和模型管理。但到目前为止，与它交互的唯一方式是 curl。本章我们添加一个 Web UI，让你能在浏览器里翻译。

完成后的界面如下。

![Web UI](../webui.png#large-center)

- 输入文本时，token 会一个个出现（带防抖）
- 可以从顶部下拉框切换模型和语言
- 选中一个尚未下载的模型时，会带进度条开始下载（可取消）

HTML、CSS 和 JavaScript 代码都很精简。我们不使用任何 CSS 框架——布局只用纯 CSS（约 100 行）。因为这是一本 C++ 的书，我们不会深入前端细节。只需告诉你「这样写，就能实现那个效果」。

## 5.1 文件结构

本章会新增以下文件。我们把 HTML、CSS 和 JavaScript 放在 `public/` 目录，由服务器提供它们。

```ascii
translate-app/
├── public/
│   ├── index.html
│   ├── style.css
│   └── script.js
└── src/
    └── main.cpp      # Add set_mount_point
```

## 5.2 设置静态文件服务

使用 cpp-httplib 的 `set_mount_point`，你可以把一个目录直接通过 HTTP 提供服务。创建 `public/` 目录，并在里面放一个空的 `index.html`。

```bash
mkdir public
```

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>Translate App</title>
</head>
<body>
  <h1>Hello!</h1>
</body>
</html>
```

在服务器代码中加一行 `set_mount_point` 并重新构建。

```cpp
// Add inside `main()`, before `svr.listen()`
svr.set_mount_point("/", "./public");
```

启动服务器，在浏览器打开 `http://127.0.0.1:8080`——你应该能看到 "Hello!"。因为这些是静态文件，修改 `index.html` 后只需刷新浏览器就能看到变化。无需重启服务器。

## 5.3 搭建布局

把 `index.html` 替换为最终的布局。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Translate App</title>
  <!-- Set favicon with inline SVG emoji (no image file needed) -->
  <link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>🌐</text></svg>">
  <link rel="stylesheet" href="/style.css">
</head>
<body>
  <!-- Header: title + model selector + language selector -->
  <header>
    <strong>Translate App</strong>
    <div>
      <!-- Options are dynamically populated by script.js via `GET /models` -->
      <select id="model-select" aria-label="Model"></select>
      <select id="target-lang" aria-label="Target language">
        <option value="ja">Japanese</option>
        <option value="en">English</option>
        <option value="zh">Chinese</option>
        <option value="ko">Korean</option>
        <option value="fr">French</option>
        <option value="de">German</option>
        <option value="es">Spanish</option>
      </select>
    </div>
  </header>

  <!-- Two-column layout: input and translation result -->
  <main>
    <textarea id="input-text" placeholder="Enter text to translate..."></textarea>
    <output id="output-text"></output>
  </main>

  <!-- Modal displayed during model download -->
  <dialog id="download-dialog">
    <h3>Downloading model...</h3>
    <progress id="download-progress" max="100" value="0"></progress>
    <p id="download-status"></p>
    <button id="download-cancel">Cancel</button>
  </dialog>

  <script src="/script.js"></script>
</body>
</html>
```

关于 HTML 的要点。

- favicon 使用内联的 SVG emoji，因而不需要图片文件
- `<dialog>` 用于显示下载进度。它是一个标准 HTML 元素，可以用 `showModal()` 作为弹窗显示
- `<output>` 用于显示翻译结果。它是一个语义上表示「计算输出」的元素
- 没有翻译按钮。输入文本时会自动开始翻译（在 5.4 节实现）

把 CSS 写到 `public/style.css`。我们不使用任何 CSS 框架——布局只用纯 CSS。

```css
:root {
  --gap: 0.5rem;
  --color-border: #ccc;
  --font: system-ui, sans-serif;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  height: 100%;
  font-family: var(--font);
}

body {
  display: flex;
  flex-direction: column;
  padding: var(--gap);
  gap: var(--gap);
}

/* Header: title + dropdowns */
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

header div {
  display: flex;
  gap: var(--gap);
}

/* Main: two-column layout */
main {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: var(--gap);
  min-height: 0;
}

#input-text {
  resize: none;
  padding: 0.75rem;
  font-family: var(--font);
  font-size: 1rem;
  border: 1px solid var(--color-border);
  border-radius: 4px;
}

textarea:focus,
select:focus {
  outline: 1px solid #4a9eff;
  outline-offset: -1px;
}

#output-text {
  display: block;
  padding: 0.75rem;
  font-size: 1rem;
  border: 1px solid var(--color-border);
  border-radius: 4px;
  white-space: pre-wrap;
  overflow-y: auto;
}

/* Download modal */
dialog {
  border: 1px solid var(--color-border);
  border-radius: 8px;
  padding: 1.5rem;
  max-width: 400px;
  width: 90%;
  margin: auto;
}

dialog::backdrop {
  background: rgba(0, 0, 0, 0.4);
}

dialog h3 {
  margin-bottom: 0.75rem;
}

dialog progress {
  width: 100%;
  height: 1.25rem;
}

dialog p {
  margin-top: 0.5rem;
  text-align: center;
  color: #666;
}

dialog button {
  display: block;
  margin: 0.75rem auto 0;
  padding: 0.4rem 1.5rem;
  cursor: pointer;
}

/* Block the entire UI during translation or model switching */
body.busy {
  cursor: wait;
}

body.busy select,
body.busy textarea {
  pointer-events: none;
  opacity: 0.6;
}
```

关于布局的要点。

- `body` 用 Flexbox 做纵向布局，`main` 用 `flex: 1` 占据剩余高度。输入区和输出区会延伸到窗口底部
- `main` 用 CSS Grid 的 `1fr 1fr` 分为两栏
- `--gap` 变量统一了所有间距。顶部、header 与盒子之间、盒子底部的宽度都一致
- `body.busy` 类在翻译或切换模型时锁定 UI。由 JavaScript 开关切换

刷新浏览器，你应该能看到输入区和输出区并排显示。现在输入还没反应，但布局已完成。

## 5.4 接上翻译功能

现在该由 JavaScript 调用服务器 API 了。创建 `public/script.js`。

### 读取 SSE 流

第 3 章构建的 `/translate/stream` 端点是一个 POST 端点。由于浏览器的 `EventSource` 只支持 GET，我们用 `fetch()` + `ReadableStream` 来读取 SSE。基本模式是：

1. 用 `fetch()` 发送 POST 请求
2. 用 `res.body.getReader()` 获取流
3. 读取块（chunk）时处理以 `data:` 开头的行

块可能在 SSE 行的中间被切开，因此我们需要缓存它们并逐行处理。

### 带防抖的自动翻译

我们没有翻译按钮，而是在输入文本或切换语言时自动触发翻译。我们加了 300ms 防抖，防止每次击键都发出请求。

为了在连续输入时取消上一次翻译，我们使用 `AbortController`。当新输入到来时，`abort()` 取消上一个 `fetch` 并开始新翻译。由于需要向 `fetch` 传入用于取消的 `signal`，SSE 读取部分写成了内联形式。

```js
const inputText = document.getElementById("input-text");
const outputText = document.getElementById("output-text");
const targetLang = document.getElementById("target-lang");

let debounceTimer = null;
let abortController = null;

async function translate() {
  const text = inputText.value.trim();
  if (!text) {
    outputText.textContent = "";
    return;
  }

  // Cancel any in-progress translation
  if (abortController) abortController.abort();
  abortController = new AbortController();
  const { signal } = abortController;

  outputText.textContent = "";
  document.body.classList.add("busy");

  try {
    const res = await fetch("/translate/stream", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text, target_lang: targetLang.value }),
      signal,
    });

    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.error || `HTTP ${res.status}`);
    }

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split("\n");
      buffer = lines.pop();

      for (const line of lines) {
        if (line.startsWith("data: ")) {
          const data = line.slice(6);
          if (data === "[DONE]") return;
          const parsed = JSON.parse(data);
          if (parsed && parsed.error) {
            outputText.textContent = "Error: " + parsed.error;
            return;
          }
          outputText.textContent += parsed;
        }
      }
    }
  } catch (e) {
    if (e.name === "AbortError") return; // Cancelled by new input
    outputText.textContent = "Error: " + e.message;
  } finally {
    document.body.classList.remove("busy");
  }
}

function scheduleTranslation() {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(translate, 300);
}

inputText.addEventListener("input", scheduleTranslation);
targetLang.addEventListener("change", scheduleTranslation);
```

我们直接使用 `fetch`，因为需要传入 `AbortController` 的 `signal`。由于服务器可能以 JSON 对象返回错误（来自第 3 章添加的 `try/catch`），我们也检查了 `parsed.error`。

刷新浏览器，试着输入一些文本。300ms 后，token 应该逐个出现。如果你修改输入，上一次翻译会取消并重新开始。

## 5.5 接上模型选择

### 加载模型列表

页面加载时，我们调用 `GET /models` 初始化下拉框。

```js
const modelSelect = document.getElementById("model-select");

// Fetch model list from `GET /models` and build the dropdown
async function loadModels() {
  const res = await fetch("/models");
  const { models } = await res.json();

  modelSelect.innerHTML = ""; // Clear existing options
  for (const m of models) {
    const opt = document.createElement("option");
    opt.value = m.name;
    // Mark undownloaded models with a ⬇ icon to distinguish them
    opt.textContent = m.downloaded
      ? `${m.name} (${m.params})`
      : `${m.name} (${m.params}) ⬇`;
    opt.selected = m.selected; // Select the current model using the `selected` flag from the server
    modelSelect.appendChild(opt);
  }
}

loadModels(); // Run on page load
```

尚未下载的模型用 `⬇` 图标加以区分。

### 切换模型

改变下拉框会调用 `POST /models/select`。如果需要下载，会弹出带进度条的 `<dialog>`。取消按钮可以中止下载。

和翻译一样，我们使用 `AbortController`。点击取消按钮会调用 `abort()` 断开连接。服务器检测到断开后中止下载（得益于第 4 章中 `download_model` 返回 `sink.os.good()`）。

```js
const dialog = document.getElementById("download-dialog");
const progressBar = document.getElementById("download-progress");
const downloadStatus = document.getElementById("download-status");
const downloadCancel = document.getElementById("download-cancel");

let modelAbort = null;

downloadCancel.addEventListener("click", () => {
  if (modelAbort) modelAbort.abort();
});

modelSelect.addEventListener("change", async () => {
  const name = modelSelect.value;
  document.body.classList.add("busy");

  modelAbort = new AbortController();
  const { signal } = modelAbort;

  try {
    const res = await fetch("/models/select", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ model: name }),
      signal,
    });

    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.error || `HTTP ${res.status}`);
    }

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split("\n");
      buffer = lines.pop();

      for (const line of lines) {
        if (line.startsWith("data: ")) {
          const data = line.slice(6);
          if (data === "[DONE]") return;
          const event = JSON.parse(data);

          switch (event.status) {
            case "downloading":
              if (!dialog.open) dialog.showModal(); // Show the modal
              progressBar.value = event.progress;   // Update the progress bar
              downloadStatus.textContent = `${event.progress}%`;
              break;
            case "loading":
              // Removing the `value` attribute puts `<progress>` into animated (indeterminate) state
              progressBar.removeAttribute("value");
              downloadStatus.textContent = "Loading model...";
              break;
            case "ready":
              if (dialog.open) dialog.close();
              break;
            case "error":
              if (dialog.open) dialog.close();
              alert("Download failed: " + event.message);
              break;
          }
        }
      }
    }

    await loadModels(); // Refresh the list since the `selected` flag changed
    scheduleTranslation(); // Re-translate with the new model
  } catch (e) {
    if (e.name === "AbortError") {
      // Cancelled -- revert to the original model
      await loadModels();
    } else {
      alert("Error: " + e.message);
    }
  } finally {
    document.body.classList.remove("busy");
    if (dialog.open) dialog.close();
    modelAbort = null;
  }
});
```

`progressBar.removeAttribute("value")` 会把 `<progress>` 元素置为不确定（动画）状态。我们在下载完成后加载模型时使用它。

## 5.6 完整代码

<details>
<summary data-file="index.html">完整代码（index.html）</summary>

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Translate App</title>
  <!-- Set favicon with inline SVG emoji (no image file needed) -->
  <link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>🌐</text></svg>">
  <link rel="stylesheet" href="/style.css">
</head>
<body>
  <!-- Header: title + model selector + language selector -->
  <header>
    <strong>Translate App</strong>
    <div>
      <!-- Options are dynamically populated by script.js via `GET /models` -->
      <select id="model-select" aria-label="Model"></select>
      <select id="target-lang" aria-label="Target language">
        <option value="ja">Japanese</option>
        <option value="en">English</option>
        <option value="zh">Chinese</option>
        <option value="ko">Korean</option>
        <option value="fr">French</option>
        <option value="de">German</option>
        <option value="es">Spanish</option>
      </select>
    </div>
  </header>

  <!-- Two-column layout: input and translation result -->
  <main>
    <textarea id="input-text" placeholder="Enter text to translate..."></textarea>
    <output id="output-text"></output>
  </main>

  <!-- Modal displayed during model download -->
  <dialog id="download-dialog">
    <h3>Downloading model...</h3>
    <progress id="download-progress" max="100" value="0"></progress>
    <p id="download-status"></p>
    <button id="download-cancel">Cancel</button>
  </dialog>

  <script src="/script.js"></script>
</body>
</html>
```

</details>

<details>
<summary data-file="style.css">完整代码（style.css）</summary>

```css
:root {
  --gap: 0.5rem;
  --color-border: #ccc;
  --font: system-ui, sans-serif;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  height: 100%;
  font-family: var(--font);
}

body {
  display: flex;
  flex-direction: column;
  padding: var(--gap);
  gap: var(--gap);
}

/* Header: title + dropdowns */
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

header div {
  display: flex;
  gap: var(--gap);
}

/* Main: two-column layout */
main {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: var(--gap);
  min-height: 0;
}

#input-text {
  resize: none;
  padding: 0.75rem;
  font-family: var(--font);
  font-size: 1rem;
  border: 1px solid var(--color-border);
  border-radius: 4px;
}

textarea:focus,
select:focus {
  outline: 1px solid #4a9eff;
  outline-offset: -1px;
}

#output-text {
  display: block;
  padding: 0.75rem;
  font-size: 1rem;
  border: 1px solid var(--color-border);
  border-radius: 4px;
  white-space: pre-wrap;
  overflow-y: auto;
}

/* Download modal */
dialog {
  border: 1px solid var(--color-border);
  border-radius: 8px;
  padding: 1.5rem;
  max-width: 400px;
  width: 90%;
  margin: auto;
}

dialog::backdrop {
  background: rgba(0, 0, 0, 0.4);
}

dialog h3 {
  margin-bottom: 0.75rem;
}

dialog progress {
  width: 100%;
  height: 1.25rem;
}

dialog p {
  margin-top: 0.5rem;
  text-align: center;
  color: #666;
}

dialog button {
  display: block;
  margin: 0.75rem auto 0;
  padding: 0.4rem 1.5rem;
  cursor: pointer;
}

/* Block the entire UI during translation or model switching */
body.busy {
  cursor: wait;
}

body.busy select,
body.busy textarea {
  pointer-events: none;
  opacity: 0.6;
}
```

</details>

<details>
<summary data-file="script.js">完整代码（script.js）</summary>

```js
// --- DOM Elements ---

const inputText = document.getElementById("input-text");
const outputText = document.getElementById("output-text");
const targetLang = document.getElementById("target-lang");
const modelSelect = document.getElementById("model-select");
const dialog = document.getElementById("download-dialog");
const progressBar = document.getElementById("download-progress");
const downloadStatus = document.getElementById("download-status");
const downloadCancel = document.getElementById("download-cancel");

// --- Model List ---

// Fetch model list from `GET /models` and build the dropdown
async function loadModels() {
  const res = await fetch("/models");
  const { models } = await res.json();

  modelSelect.innerHTML = ""; // Clear existing options
  for (const m of models) {
    const opt = document.createElement("option");
    opt.value = m.name;
    // Mark undownloaded models with a ⬇ icon to distinguish them
    opt.textContent = m.downloaded
      ? `${m.name} (${m.params})`
      : `${m.name} (${m.params}) ⬇`;
    opt.selected = m.selected; // Select the current model using the `selected` flag from the server
    modelSelect.appendChild(opt);
  }
}

loadModels(); // Run on page load

// --- Translation (auto-translation with debounce) ---

let debounceTimer = null;
let abortController = null;

async function translate() {
  const text = inputText.value.trim();
  if (!text) {
    outputText.textContent = "";
    return;
  }

  // Cancel any in-progress translation
  if (abortController) abortController.abort();
  abortController = new AbortController();
  const { signal } = abortController;

  outputText.textContent = "";
  document.body.classList.add("busy");

  try {
    const res = await fetch("/translate/stream", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text, target_lang: targetLang.value }),
      signal,
    });

    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.error || `HTTP ${res.status}`);
    }

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split("\n");
      buffer = lines.pop();

      for (const line of lines) {
        if (line.startsWith("data: ")) {
          const data = line.slice(6);
          if (data === "[DONE]") return;
          const parsed = JSON.parse(data);
          if (parsed && parsed.error) {
            outputText.textContent = "Error: " + parsed.error;
            return;
          }
          outputText.textContent += parsed;
        }
      }
    }
  } catch (e) {
    if (e.name === "AbortError") return; // Cancelled by new input
    outputText.textContent = "Error: " + e.message;
  } finally {
    document.body.classList.remove("busy");
  }
}

function scheduleTranslation() {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(translate, 300);
}

inputText.addEventListener("input", scheduleTranslation);
targetLang.addEventListener("change", scheduleTranslation);

// --- Model Selection ---

let modelAbort = null;

downloadCancel.addEventListener("click", () => {
  if (modelAbort) modelAbort.abort();
});

modelSelect.addEventListener("change", async () => {
  const name = modelSelect.value;
  document.body.classList.add("busy");

  modelAbort = new AbortController();
  const { signal } = modelAbort;

  try {
    const res = await fetch("/models/select", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ model: name }),
      signal,
    });

    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.error || `HTTP ${res.status}`);
    }

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split("\n");
      buffer = lines.pop();

      for (const line of lines) {
        if (line.startsWith("data: ")) {
          const data = line.slice(6);
          if (data === "[DONE]") return;
          const event = JSON.parse(data);

          switch (event.status) {
            case "downloading":
              if (!dialog.open) dialog.showModal();
              progressBar.value = event.progress;
              downloadStatus.textContent = `${event.progress}%`;
              break;
            case "loading":
              progressBar.removeAttribute("value");
              downloadStatus.textContent = "Loading model...";
              break;
            case "ready":
              if (dialog.open) dialog.close();
              break;
            case "error":
              if (dialog.open) dialog.close();
              alert("Download failed: " + event.message);
              break;
          }
        }
      }
    }

    await loadModels();
    scheduleTranslation(); // Re-translate with the new model
  } catch (e) {
    if (e.name === "AbortError") {
      // Cancelled -- revert to the original model
      await loadModels();
    } else {
      alert("Error: " + e.message);
    }
  } finally {
    document.body.classList.remove("busy");
    if (dialog.open) dialog.close();
    modelAbort = null;
  }
});
```

</details>

<details>
<summary data-file="main.cpp">完整代码（main.cpp）</summary>

服务器端唯一的改动就是那一行 `set_mount_point`。在第 4 章的完整代码中，把它加在 `svr.listen()` 之前。

```cpp
#include <httplib.h>
#include <nlohmann/json.hpp>
#include <cpp-llamalib.h>

#include <algorithm>
#include <csignal>
#include <filesystem>
#include <fstream>
#include <iostream>

using json = nlohmann::json;

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
  cli.set_follow_location(true);  // Hugging Face redirects to CDN
  cli.set_read_timeout(std::chrono::hours(1)); // Long timeout for large models

  auto url = "/" + model.repo + "/resolve/main/" + model.filename;
  auto path = get_models_dir() / model.filename;
  auto tmp_path = std::filesystem::path(path).concat(".tmp");

  std::ofstream ofs(tmp_path, std::ios::binary);
  if (!ofs) { return false; }

  auto res = cli.Get(url,
    // content_receiver: receive chunks and write to file
    [&](const char *data, size_t len) {
      ofs.write(data, len);
      return ofs.good();
    },
    // progress: report download progress (return false to abort)
    [&, last_pct = -1](size_t current, size_t total) mutable {
      int pct = total ? (int)(current * 100 / total) : 0;
      if (pct == last_pct) return true; // Skip if same value
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

httplib::Server svr;

void signal_handler(int sig) {
  if (sig == SIGINT || sig == SIGTERM) {
    std::cout << "\nReceived signal, shutting down gracefully...\n";
    svr.stop();
  }
}

int main() {
  // Create model storage directory
  auto models_dir = get_models_dir();
  std::filesystem::create_directories(models_dir);

  // Auto-download default model if not present
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

  // LLM inference takes time, so set a longer timeout (default is 5 seconds)
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
      auto translation = llm.chat(prompt);
      res.set_content(json{{"translation", translation}}.dump(),
                      "application/json");
    } catch (const std::exception &e) {
      res.status = 500;
      res.set_content(json{{"error", e.what()}}.dump(), "application/json");
    }
  });

  // --- SSE streaming translation (Chapter 3) --------------------------------

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

  // --- Model list (Chapter 4) -----------------------------------------------

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

  // --- Model selection (Chapter 4) ------------------------------------------

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
          // SSE event send helper
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
          llm = llamalib::Llama{path};
          selected_model = model.filename;

          send({{"status", "ready"}});
          sink.done();
          return true;
        });
  });

  // --- Static file serving (Chapter 5) --------------------------------------

  svr.set_mount_point("/", "./public");

  // Allow graceful shutdown via `Ctrl+C` (`SIGINT`) or `kill` (`SIGTERM`)
  signal(SIGINT, signal_handler);
  signal(SIGTERM, signal_handler);

  std::cout << "Listening on http://127.0.0.1:8080" << std::endl;
  svr.listen("127.0.0.1", 8080);
}
```

</details>

## 5.7 测试

重编并启动服务器。

```bash
cmake --build build -j
./build/translate-server
```

在浏览器中打开 `http://127.0.0.1:8080`。

1. 输入一些文本——300ms 后，token 逐个出现
2. 修改输入——上一次翻译被取消，重新开始
3. 切换语言下拉框——自动重新翻译
4. 切换模型下拉框——已下载时立即切换
5. 选择一个未下载的模型——出现进度条，可用 Cancel 中止

第 4 章用 curl 做的所有事，现在都能在浏览器里完成了。

## 下一章

服务器和 Web UI 都已经完成。下一章，我们用 webview/webview 包装这个应用，把它变成无需浏览器即可运行的桌面应用。我们会把静态文件嵌入二进制，使分发产物成为一个单独的可执行文件。

**下一章：**[用 WebView 变成桌面应用](../ch06-desktop-app)
