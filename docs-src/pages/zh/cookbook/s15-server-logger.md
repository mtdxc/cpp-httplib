---
title: "S15. 在服务端记录请求日志"
order: 34
status: "draft"
---

要记录服务端接收的请求和返回的响应，使用 `Server::set_logger()`。回调会在每个完成的请求上触发一次，使其成为访问日志和指标采集的基础。

## 基本用法

```cpp
svr.set_logger([](const httplib::Request &req, const httplib::Response &res) {
  std::cout << req.remote_addr << " "
            << req.method << " " << req.path
            << " -> " << res.status << std::endl;
});
```

日志回调同时接收 `Request` 和 `Response`。你可以抓取方法、路径、状态码、客户端 IP、请求头、请求体——任何你需要的。

## 访问日志风格的格式

下面是一个类似 Apache/Nginx 的访问日志格式。

```cpp
svr.set_logger([](const auto &req, const auto &res) {
  auto now = std::time(nullptr);
  char timebuf[32];
  std::strftime(timebuf, sizeof(timebuf), "%Y-%m-%d %H:%M:%S",
                std::localtime(&now));

  std::cout << timebuf << " "
            << req.remote_addr << " "
            << "\"" << req.method << " " << req.path << "\" "
            << res.status << " "
            << res.body.size() << "B"
            << std::endl;
});
```

## 度量请求耗时

要在日志中包含请求时长，从一个 pre-routing 处理函数把一个起始时间戳存进 `res.user_data`，然后在日志器里相减。

```cpp
svr.set_pre_routing_handler([](const auto &req, auto &res) {
  res.user_data.set("start", std::chrono::steady_clock::now());
  return httplib::Server::HandlerResponse::Unhandled;
});

svr.set_logger([](const auto &req, const auto &res) {
  auto *start = res.user_data.get<std::chrono::steady_clock::time_point>("start");
  auto elapsed = start
    ? std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - *start).count()
    : 0;
  std::cout << req.method << " " << req.path
            << " " << res.status << " " << elapsed << "ms" << std::endl;
});
```

关于 `user_data` 的更多内容，参见 [S12. 用 `res.user_data` 在处理函数间传递数据](../s12-user-data)。

> **提示：** 日志器在与请求处理相同的线程上同步运行。其中的重活会损害吞吐量——如果你需要执行耗时的操作，把它投递到队列并异步处理。
