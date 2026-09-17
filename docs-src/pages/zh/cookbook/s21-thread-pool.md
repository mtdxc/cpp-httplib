---
title: "S21. 配置线程池"
order: 40
status: "draft"
---

cpp-httplib 从一个线程池提供服务请求。默认情况下，基础线程数是 `std::thread::hardware_concurrency() - 1` 和 `8` 中的较大者，并且它能动态扩容到那个值的 4 倍。要明确设置线程数，通过 `new_task_queue` 提供你自己的工厂。

## 设置线程数

```cpp
httplib::Server svr;

svr.new_task_queue = [] {
  return new httplib::ThreadPool(/*base_threads=*/8, /*max_threads=*/64);
};

svr.listen("0.0.0.0", 8080);
```

工厂是一个返回 `TaskQueue*` 的 lambda。把 `base_threads` 和 `max_threads` 传给 `ThreadPool`，池会根据负载在其间伸缩。空闲线程在超时后退出（默认 3 秒）。

## 也限制队列

如果不加限制地增长，待处理队列会吞噬内存。你也可以给它封顶。

```cpp
svr.new_task_queue = [] {
  return new httplib::ThreadPool(
    /*base_threads=*/12,
    /*max_threads=*/0,   // disable dynamic scaling
    /*max_queued_requests=*/18);
};
```

`max_threads=0` 禁用动态伸缩——你得到一个固定的 `base_threads`。放不进 `max_queued_requests` 的请求会被拒绝。

## 使用你自己的线程池

你可以通过继承 `TaskQueue` 并从工厂返回它，接入一个完全自定义的线程池。

```cpp
class MyTaskQueue : public httplib::TaskQueue {
public:
  MyTaskQueue(size_t n) { pool_.start_with_thread_count(n); }
  bool enqueue(std::function<void()> fn) override { return pool_.post(std::move(fn)); }
  void shutdown() override { pool_.shutdown(); }

private:
  MyThreadPool pool_;
};

svr.new_task_queue = [] { return new MyTaskQueue(12); };
```

当你的项目里已有一个线程池、想把线程管理统一起来时，这很方便。

## 编译期调优

如果你想做编译期配置，可以用宏设置默认值。

```cpp
#define CPPHTTPLIB_THREAD_POOL_COUNT 16       // base thread count
#define CPPHTTPLIB_THREAD_POOL_MAX_COUNT 128   // max thread count
#define CPPHTTPLIB_THREAD_POOL_IDLE_TIMEOUT 5  // seconds before idle threads exit
#include <httplib.h>
```

> **提示：** 一条 WebSocket 连接会在其整个生命周期内占用一个工作线程。对于大量并发的 WebSocket 连接，启用动态伸缩（例如 `ThreadPool(8, 64)`）。
