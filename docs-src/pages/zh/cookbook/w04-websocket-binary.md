---
title: "W04. 发送与接收二进制帧"
order: 55
status: "draft"
---

WebSocket 有两种帧类型：文本和二进制。JSON 和纯文本放入文本帧；图片和原始协议字节放入二进制帧。在 cpp-httplib 中，`send()` 通过重载选择正确的类型。

## 如何选择帧类型

```cpp
ws.send(std::string("Hello"));           // text
ws.send("Hello", 5);                      // binary
ws.send(binary_data, binary_data_size);   // binary
```

`std::string` 重载以**文本**发送；`const char*` + 大小的重载以**二进制**发送。有点微妙，但一旦知道就记得住。

如果你有一个 `std::string` 并想以二进制发送它，显式传入 `.data()` 和 `.size()`。

```cpp
std::string raw = build_binary_payload();
ws.send(raw.data(), raw.size()); // binary frame
```

## 接收时检测帧类型

`ws.read()` 的返回值会告诉你收到的帧是文本还是二进制。

```cpp
std::string msg;
auto result = ws.read(msg);

switch (result) {
  case httplib::ws::ReadResult::Text:
    std::cout << "text: " << msg << std::endl;
    break;
  case httplib::ws::ReadResult::Binary:
    std::cout << "binary: " << msg.size() << " bytes" << std::endl;
    handle_binary(msg.data(), msg.size());
    break;
  case httplib::ws::ReadResult::Fail:
    // error or closed
    break;
  case httplib::ws::ReadResult::Timeout:
    // read timeout elapsed; the connection is still open
    break;
}
```

二进制帧仍然以一个 `std::string` 返回，但要把其内容当作原始字节对待——使用 `msg.data()` 和 `msg.size()`。

## 何时二进制是正确的选择

- **图片、视频、音频**：没有 Base64 开销
- **自定义协议**：protobuf、MessagePack，或任意结构化的二进制格式
- **游戏网络**：当延迟很重要时
- **传感器数据流**：直接推送数值数组

## Ping 略带二进制色彩，但被隐藏

在操作码层面，WebSocket 的 Ping/Pong 帧与二进制帧是近亲，但 cpp-httplib 会自动处理它们——你无需触碰。参见 [W02. 设置 WebSocket 心跳](../w02-websocket-ping)。

## 示例：发送一张图片

```cpp
// Server: push an image
svr.WebSocket("/image", [](const auto &req, auto &ws) {
  auto img = read_image_file("logo.png");
  ws.send(img.data(), img.size());
});
```

```cpp
// Client: receive and save
httplib::ws::WebSocketClient cli("ws://localhost:8080/image");
cli.connect();

std::string buf;
if (cli.read(buf) == httplib::ws::ReadResult::Binary) {
  std::ofstream ofs("received.png", std::ios::binary);
  ofs.write(buf.data(), buf.size());
}
```

你可以在同一个连接里混合文本和二进制。一个常见的模式：用 JSON 做控制消息，用二进制做实际数据——元数据和载荷都能得到高效处理。

> **提示：** WebSocket 帧的大小限制并非无限。对于非常大的数据，在你的应用代码里将其分块。cpp-httplib 能一次性处理一个大帧，但它会一次性把它们全加载到内存中。
