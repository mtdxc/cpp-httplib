---
title: "S06. 返回文件下载响应"
order: 25
status: "draft"
---

要强制浏览器弹出**下载对话框**而不是内联渲染，发送一个 `Content-Disposition` 请求头。为此cpp-httplib没有提供特殊的API——它只是一个请求头。

## 基本用法

```cpp
svr.Get("/download/report", [](const httplib::Request &req, httplib::Response &res) {
  res.set_header("Content-Disposition", "attachment; filename=\"report.pdf\"");
  res.set_file_content("reports/2026-04.pdf", "application/pdf");
});
```

`Content-Disposition: attachment` 会让浏览器弹出“另存为”对话框。`filename=` 参数会成为默认的保存名称。

## 非 ASCII 文件名

对于包含非 ASCII 字符或空格的文件名，使用 RFC 5987 的 `filename*` 形式。

```cpp
svr.Get("/download/report", [](const httplib::Request &req, httplib::Response &res) {
  res.set_header(
    "Content-Disposition",
    "attachment; filename=\"report.pdf\"; "
    "filename*=UTF-8''%E3%83%AC%E3%83%9D%E3%83%BC%E3%83%88.pdf");
  res.set_file_content("reports/2026-04.pdf", "application/pdf");
});
```

`filename*=UTF-8''` 后面的部分是 URL 编码的 UTF-8。保留 ASCII 的 `filename=` 作为旧浏览器的回退。

## 下载动态生成的数据

你不需要一个真实文件——你可以直接把一个生成的字符串作为下载提供。

```cpp
svr.Get("/export.csv", [](const httplib::Request &req, httplib::Response &res) {
  std::string csv = build_csv();
  res.set_header("Content-Disposition", "attachment; filename=\"export.csv\"");
  res.set_content(csv, "text/csv");
});
```

这是 CSV 导出的经典写法。

> **提示：** 某些浏览器仅根据 Content-Type 就会触发下载，即使没有 `Content-Disposition`。相反，设置为 `inline` 会尽可能在浏览器中渲染内容。
