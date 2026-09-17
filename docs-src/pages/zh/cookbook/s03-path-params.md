---
title: "S03. 使用路径参数"
order: 22
status: "draft"
---

对于像 `/users/:id` 这样的动态 URL——REST API 的标配——只需在路径模式里放 `:name`。匹配到的值会落在 `req.path_params` 中。

## 基本用法

```cpp
svr.Get("/users/:id", [](const httplib::Request &req, httplib::Response &res) {
  auto id = req.path_params.at("id");
  res.set_content("user id: " + id, "text/plain");
});
```

对 `/users/42` 的请求会把 `req.path_params["id"]` 填为 `"42"`。`path_params` 是一个 `std::unordered_map<std::string, std::string>`，所以用 `at()` 来读取它。

## 多个参数

你需要多少就可以有多少。

```cpp
svr.Get("/orgs/:org/repos/:repo", [](const httplib::Request &req, httplib::Response &res) {
  auto org = req.path_params.at("org");
  auto repo = req.path_params.at("repo");
  res.set_content(org + "/" + repo, "text/plain");
});
```

这会匹配像 `/orgs/anthropic/repos/cpp-httplib` 这样的路径。

## 正则模式

要更灵活的匹配，使用基于 `std::regex` 的模式。

```cpp
svr.Get(R"(/users/(\d+))", [](const httplib::Request &req, httplib::Response &res) {
  auto id = req.matches[1];
  res.set_content("user id: " + std::string(id), "text/plain");
});
```

模式中的括号会成为 `req.matches` 中的捕获组。`req.matches[0]` 是完整匹配；`req.matches[1]` 及以后是捕获组。

## 用哪个

- 对于普通的 ID 或 slug，`:name` 就够了——可读，且形状一目了然
- 当你想把 URL 限制为比如仅数字或 UUID 格式时，用正则
- 混用两者可能会令人困惑——每个项目坚持一种风格为好

> **提示：** 路径参数以字符串形式传入。如果你需要一个整数，用 `std::stoi()` 转换，别忘了处理转换错误。
