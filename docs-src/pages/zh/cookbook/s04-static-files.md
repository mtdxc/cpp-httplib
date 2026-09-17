---
title: "S04. 提供静态文件服务"
order: 23
status: "draft"
---

要提供像 HTML、CSS 和图片这样的静态文件，使用 `set_mount_point()`。只需把一个 URL 路径映射到一个本地目录，整个目录就会变得可访问。

## 基本用法

```cpp
httplib::Server svr;
svr.set_mount_point("/", "./public");
svr.listen("0.0.0.0", 8080);
```

`./public/index.html` 现在可以通过 `http://localhost:8080/index.html` 访问，`./public/css/style.css` 可以通过 `http://localhost:8080/css/style.css` 访问。目录布局直接映射到 URL。

## 多个挂载点

你可以注册多个挂载点。

```cpp
svr.set_mount_point("/", "./public");
svr.set_mount_point("/assets", "./dist/assets");
svr.set_mount_point("/uploads", "./var/uploads");
```

你甚至可以在同一路径上挂载多个目录——它们按注册顺序搜索，第一个命中者获胜。

## 与 API 处理函数配合使用

静态文件和 API 处理函数可以愉快共存。用 `Get()` 及同类函数注册的处理函数优先；只有当没匹配时才会搜索挂载点。

```cpp
svr.Get("/api/users", [](const auto &req, auto &res) {
  res.set_content("[]", "application/json");
});

svr.set_mount_point("/", "./public");
```

这给了你一个对 SPA 友好的布局：`/api/*` 命中处理函数，其他一切都从 `./public/` 提供。

## 添加 MIME 类型

cpp-httplib 内置一个扩展名到 Content-Type 的映射，但你也可以添加自己的。

```cpp
svr.set_file_extension_and_mimetype_mapping("wasm", "application/wasm");
```

> **警告：** 静态文件服务方法是**非线程安全的**。不要在 `listen()` 之后调用它们——应在启动服务端前把一切都配置好。

> 关于下载类型的响应，参见 [S06. 返回文件下载响应](../s06-download-response)。
