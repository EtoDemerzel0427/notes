---
title: "mlog.hpp Reference Solution"
slug: "mlog-reference-solution"
date: "2026-09-21"
tags: ["Interview", "C++", "Low Latency"]
category: "Interview Prep"
draft: false
---

这是 [[Low-Latency Logging for Options Market Making]] 一题的完整参考实现（Claude Fable 5.1 给出），约 400 行的单头文件。**它是终点，不是面试时要默写的东西**——怎么一步步走到这里见主页面。

- 编译：`g++ -std=c++20 -O2 -pthread`
- 我本地复验：3 个线程各 2 万条 `LOG_INFO`（含 `double`/`int`/`std::string`/`int` 四个参数），60001 行全部落盘、线程内顺序正确；ThreadSanitizer 无报告（注意：该次运行没有触发丢弃路径，`dropped_` 的 race 见文末）。
- 原作者实测：热路径相邻两条日志时间戳相差约 40ns；console sink 跟不上时正确触发丢弃计数（20 万条 bench 只落了 3 万条，热路径无抖动）。

```text
callsite --(memcpy)--> per-thread SPSC ring --(consumer thread)--> format --> sinks
```

## 模块导读

| 模块 | 职责 | 关键设计 |
| --- | --- | --- |
| `Level` / `MLOG_MIN_LEVEL` | 两层 level 控制 | 编译期 `if constexpr` 裁剪 + 运行期 `atomic<Level>` relaxed load |
| `Sink` / `ConsoleSink` / `FileSink` | 输出 | 只在消费者线程调用 → 无锁；`FileSink` 攒 32KB 批量 `write`；per-sink `minLevel` |
| `argSize` / `encodeArg` / `appendArg` | 参数编解码 | 热路径只有 `memcpy`；string-like **按值拷贝**（长度前缀 + 内容）；冷路径 `to_chars` |
| `formatRecord<Ts...>` / `SourceMeta` | 类型擦除的恢复 | 每种参数类型列表实例化一个格式化函数，指针存进 per-callsite 的 static；热路径只写一个 `SourceMeta*` |
| `SpscRing` | 每线程一个字节 ring | `reserve`/`commit` 两阶段；记录不跨末尾（padding header）；index cache；满了返回 `nullptr` 并计数 |
| `Logger::log` | 热路径 | 算大小 → reserve → 写 header → fold expression 编码参数 → commit |
| `localRing` / `ThreadHandle` | 线程注册与退出 | `thread_local` 惰性注册（一生只拿一次锁）；析构置 `closed`，消费者排空后才回收 |
| `consumerLoop` | 冷路径 | k-way merge（按头部时间戳取最小）→ 格式化 → dispatch；汇报丢弃数；回收已关闭队列；100ms flush；空闲 `sleep_for(50µs)` |
| `MLOG` 宏 | 用户接口 | 立即调用的 generic lambda 提供 per-callsite static 和参数类型；参数惰性求值 |

## 同步原语清单：这里面的 mutex / atomic 都是干什么的

"无锁 logger"不等于代码里没有锁，而是**热路径上没有锁**。逐个看：

| 原语 | 保护/传递什么 | 谁在什么时候碰它 | 在热路径上吗 |
| --- | --- | --- | --- |
| `Logger::mu_`（`std::mutex`） | `queues_`（ring 的注册表）和 `sinks_` | ① `addSink`：启动时；② `localRing()`：**每个线程第一次打日志时一次**，注册自己的 ring；③ 消费者每轮 `drainOnce` 开头拷贝一份 `queues_`、结尾回收已关闭的 ring；④ `dispatch` / `flushAll`：消费者每行日志一次（不必要，已知问题 #1） | ❌ 每线程一生只有一次；其余全在消费者线程或启动阶段 |
| `SpscRing::tail_`（`atomic<size_t>`） | 生产者 → 消费者："写到哪了" | 生产者 `commit` 时 store（`release`）；消费者发现"看起来空了"时 load（`acquire`） | ✅ 一次普通 `mov` |
| `SpscRing::head_`（`atomic<size_t>`） | 消费者 → 生产者："读到哪了" | 消费者 `pop` 时 store；生产者发现"看起来满了"时 load | ✅ 但因为有 `cachedHead_`，几乎从不真的去读 |
| `cachedHead_` / `cachedTail_`（普通变量） | 对方索引的本地缓存 | 各自只被一个线程访问 → 不需要原子 | ✅ 纯本地读写 |
| `dropped_`（普通 `uint64_t`） | 丢弃计数 | 生产者写、消费者读 → **这个应该是 atomic**（已知问题 #6） | 仅在丢弃时 |
| `Logger::level_`（`atomic<Level>`） | 运行期 level 阈值 | 任何线程 `setLevel`；每次打日志 `enabled()` 读一次（`relaxed`） | ✅ 一次 relaxed load |
| `Logger::running_`（`atomic<bool>`） | 消费者线程的启停标志 | `start`/`stop` 用 `exchange` 保证幂等；消费者循环里 relaxed load | ❌ |
| `ThreadQueue::closed`（`atomic<bool>`） | "拥有这个 ring 的线程已退出" | 线程退出时 `ThreadHandle` 析构置 true；消费者回收前检查 | ❌ |
| `shared_ptr<ThreadQueue>` 的引用计数 | ring 的生命周期 | 线程的 `ThreadHandle` 和消费者的 `queues_` 各持一份；引用计数本身是原子的 | ❌ 热路径通过 `thread_local` 拿到引用，不拷贝 `shared_ptr` |
| `thread_local ThreadHandle` 的初始化 guard | 每线程惰性初始化 | 编译器生成；每次 `localRing()` 检查一次 | ✅ ~1ns（已知问题 #9） |
| `Logger::instance()` 的 static guard | Meyers singleton 的线程安全初始化 | 编译器生成；每次调用检查一次 | ✅ ~1ns |

**所以一条日志在热路径上实际碰到的同步操作只有**：一次 `level_` 的 relaxed load + 两次 guard 检查 + 一次 `tail_` 的 store。没有锁、没有 CAS、没有 `lock` 前缀的指令。

`mu_` 体现的是一个通用原则：**锁不是敌人，热路径上的锁才是**。注册、配置、启停这些冷路径用一把朴素的 mutex 是最好的选择——简单、正确、容易审查。更多模式见 [[Thread Synchronization Patterns in C++]]。

## 已知问题

作者自己指出的：

1. `dispatch` / `flushAll` 每次都拿 `mu_` 保护 `sinks_`，正确但不必要；应改为启动后 sinks 只读。
2. 只支持 `{}` 占位符，没有格式说明符；产线会换成 `fmt`，或走二进制日志 + 离线解码（NanoLog 路线）。
3. 没有 crash handler（SIGSEGV 时 dump 所有 ring）。
4. 消费者对线程数线性 peek；线程多时应改成堆或 batch 合并。
5. string-like 参数没有单条上限与截断。

我 review 时补充的：

6. **`SpscRing::dropped_` 是 data race**：普通 `uint64_t`，生产者写、消费者读，无同步 → 形式上是 UB。改成 `std::atomic<uint64_t>` + `store(load(relaxed) + 1, relaxed)`，单写者不需要 `lock` 前缀，零额外成本。
7. **k-way merge 不保证全局有序**：时间戳在 `commit` 之前取，某线程的 ring 此刻为空不代表它之后不会出现更早的时间戳。需要 watermark（只输出 `ts < now - δ` 的记录）。
8. **字符串长度算了两次**（`argSize` 与 `encodeArg`）：`const char*` 会 `strlen` 两遍；若期间字符串变长，第二次 `memcpy` 会越过 reserve 的范围。应只算一次并传递下去。
9. `localRing()` 的 `thread_local` 有动态初始化 → 每次访问有 guard 检查；`Logger::instance()` 同理。可用 `constinit thread_local SpscRing*` 缓存指针。
10. 每个线程首条日志会触发 1MiB 分配 + 加锁注册 + 缺页，应提供预热接口在启动阶段付掉。
11. `SourceMeta` 的 static 可以写成 `static constexpr`，强制常量初始化、确保没有 guard。
12. 头文件用了 `std::snprintf`/`gmtime_r` 但没有显式 `#include <cstdio>`/`<ctime>`（靠传递包含）。
13. **Level 和 sink 应由 config 驱动**（需求修订后）：代码里 `MLOG_MIN_LEVEL` 把 level 和编译绑定，`addSink(std::make_unique<FileSink>(...))` 把输出目标写死在调用方。应改成：启动时读 config → `setLevel` → 按 sink 类型注册表装配 → `start`；`MLOG_MIN_LEVEL` 降级为可选优化（默认 `Trace`，即不裁剪）。宏的惰性求值价值不受影响——`enabled(lvl)` 为假时参数照样不求值。
14. 空转用 `sleep_for(50µs)` 写死；应做成配置项（quill 的 `sleep_duration` 默认 500ns，0 = busy-spin），并支持 pin 核。
15. 时间戳用 `system_clock::now()`（vDSO，~20ns）；进一步优化是热路径存裸 `rdtsc`、后台校准换算，见主页面 §9.1。

## 完整代码

```cpp
// mlog.hpp — a low-latency, multi-threaded logger for a market-making system.
//
// Assumptions (clarified up front):
//   * Hot path budget ~100ns: no allocation, no formatting, no locks, no syscalls.
//   * Losing log lines is acceptable when a producer outruns the consumer;
//     we drop + count rather than block a trading thread.
//   * Per-thread ordering is guaranteed; cross-thread ordering is "best effort"
//     via a k-way merge on timestamp in the consumer.
//   * Log arguments are arithmetic types, enums, or string-like (copied by value).
//   * Compile-time level cut-off via MLOG_MIN_LEVEL, runtime cut-off via atomic.
//
// Architecture:
//   callsite --(memcpy)--> per-thread SPSC ring --(background thread)--> format --> sinks
#pragma once

#include <atomic>
#include <charconv>
#include <chrono>
#include <cstddef>
#include <cstdint>
#include <cstring>
#include <fstream>
#include <iostream>
#include <memory>
#include <mutex>
#include <sstream>
#include <string>
#include <string_view>
#include <thread>
#include <type_traits>
#include <vector>

namespace mlog {

// ───────────────────────────────── Levels ─────────────────────────────────
enum class Level : uint8_t { Trace, Debug, Info, Warn, Error, Off };

inline constexpr std::string_view levelName(Level l) {
    constexpr std::string_view names[] = {"TRACE", "DEBUG", "INFO ", "WARN ", "ERROR", "OFF  "};
    return names[static_cast<uint8_t>(l)];
}

#ifndef MLOG_MIN_LEVEL
#define MLOG_MIN_LEVEL ::mlog::Level::Trace   // -DMLOG_MIN_LEVEL=::mlog::Level::Info in release
#endif

// ───────────────────────────────── Sinks ──────────────────────────────────
// Sinks run only on the consumer thread, so they need no internal locking.
struct Sink {
    virtual ~Sink() = default;
    virtual void write(Level lvl, std::string_view line) = 0;
    virtual void flush() {}
    Level minLevel = Level::Trace;   // per-sink filtering
};

class ConsoleSink final : public Sink {
public:
    void write(Level, std::string_view line) override { std::cout << line << '\n'; }
    void flush() override { std::cout.flush(); }
};

class FileSink final : public Sink {
public:
    explicit FileSink(const std::string& path) : out_(path, std::ios::app) {
        buf_.reserve(1 << 16);
    }
    void write(Level, std::string_view line) override {
        buf_.append(line).push_back('\n');
        if (buf_.size() > (1 << 15)) flush();   // batch syscalls
    }
    void flush() override {
        if (!buf_.empty()) { out_.write(buf_.data(), static_cast<std::streamsize>(buf_.size())); buf_.clear(); }
        out_.flush();
    }
private:
    std::ofstream out_;
    std::string   buf_;
};

// ───────────────────────── Argument encode / decode ───────────────────────
// Hot path: compute size + memcpy. Cold path: decode + to_chars.
template <class T>
inline constexpr bool isStringLike =
    std::is_convertible_v<T, std::string_view> && !std::is_arithmetic_v<T>;

template <class T>
inline constexpr bool isSupportedArg =
    isStringLike<T> || std::is_arithmetic_v<T> || std::is_enum_v<T>;

template <class T>
inline size_t argSize(const T& v) {
    static_assert(isSupportedArg<T>, "unsupported log argument type");
    if constexpr (isStringLike<T>) return sizeof(uint32_t) + std::string_view(v).size();
    else                           return sizeof(T);
}

template <class T>
inline std::byte* encodeArg(std::byte* p, const T& v) {
    if constexpr (isStringLike<T>) {
        std::string_view sv(v);
        uint32_t n = static_cast<uint32_t>(sv.size());
        std::memcpy(p, &n, sizeof n);           p += sizeof n;
        std::memcpy(p, sv.data(), n);           p += n;
    } else {
        std::memcpy(p, &v, sizeof v);           p += sizeof v;
    }
    return p;
}

// Appends the text of the next argument to `out`, advancing `p`.
template <class T>
inline void appendArg(std::string& out, const std::byte*& p) {
    if constexpr (isStringLike<T>) {
        uint32_t n; std::memcpy(&n, p, sizeof n); p += sizeof n;
        out.append(reinterpret_cast<const char*>(p), n); p += n;
    } else {
        T v; std::memcpy(&v, p, sizeof v); p += sizeof v;
        if constexpr (std::is_same_v<T, bool>) {
            out += v ? "true" : "false";
        } else if constexpr (std::is_same_v<T, char>) {
            out += v;
        } else if constexpr (std::is_enum_v<T>) {
            char tmp[24]; auto r = std::to_chars(tmp, tmp + sizeof tmp, static_cast<std::underlying_type_t<T>>(v));
            out.append(tmp, r.ptr);
        } else {
            char tmp[64]; auto r = std::to_chars(tmp, tmp + sizeof tmp, v);
            out.append(tmp, r.ptr);
        }
    }
}

// Copies `fmt` up to the next "{}" into out, advances fmt past it.
inline void appendUntilBrace(std::string& out, const char*& fmt) {
    while (*fmt) {
        if (fmt[0] == '{' && fmt[1] == '}') { fmt += 2; return; }
        out += *fmt++;
    }
}

// One instantiation per unique argument-type list; pointer stored in SourceMeta.
template <class... Ts>
void formatRecord(std::string& out, const char* fmt, const std::byte* p) {
    ((appendUntilBrace(out, fmt), appendArg<Ts>(out, p)), ...);
    out.append(fmt);   // trailing text
}

using FormatFn = void (*)(std::string&, const char*, const std::byte*);

// Static, per-callsite. The hot path stores only a pointer to this.
struct SourceMeta {
    const char* fmt;
    const char* file;
    int         line;
    Level       level;
    FormatFn    format;
};

// ───────────────────────────── Ring buffer ────────────────────────────────
struct RecordHeader {
    uint64_t          ts;        // ns since epoch (swap for rdtsc if needed)
    const SourceMeta* meta;      // nullptr == padding, jump to start of buffer
    uint32_t          argBytes;
};

// Single-producer / single-consumer byte ring. Records never straddle the end:
// if a record doesn't fit contiguously we emit padding and wrap.
class SpscRing {
public:
    explicit SpscRing(size_t capacityPow2)
        : cap_(capacityPow2), mask_(capacityPow2 - 1), buf_(new std::byte[capacityPow2]) {}

    // Producer side. Returns nullptr if there is no room (caller drops).
    std::byte* reserve(size_t bytes, size_t& reserved) {
        const size_t t      = tail_.load(std::memory_order_relaxed);
        const size_t pos    = t & mask_;
        const size_t contig = cap_ - pos;
        const bool   wrap   = bytes > contig;
        reserved            = wrap ? contig + bytes : bytes;

        if (cap_ - (t - cachedHead_) < reserved) {
            cachedHead_ = head_.load(std::memory_order_acquire);   // refresh once, then retry
            if (cap_ - (t - cachedHead_) < reserved) { ++dropped_; return nullptr; }
        }
        if (wrap) {
            if (contig >= sizeof(RecordHeader)) {
                RecordHeader pad{0, nullptr, 0};
                std::memcpy(buf_.get() + pos, &pad, sizeof pad);
            }
            return buf_.get();          // record starts at offset 0
        }
        return buf_.get() + pos;
    }
    void commit(size_t reserved) {
        tail_.store(tail_.load(std::memory_order_relaxed) + reserved, std::memory_order_release);
    }

    // Consumer side. Returns pointer to next record header or nullptr if empty.
    const std::byte* peek() {
        for (;;) {
            const size_t h = head_.load(std::memory_order_relaxed);
            if (h == cachedTail_) {
                cachedTail_ = tail_.load(std::memory_order_acquire);
                if (h == cachedTail_) return nullptr;
            }
            const size_t pos = h & mask_;
            if (cap_ - pos < sizeof(RecordHeader)) {   // producer skipped without a pad header
                head_.store(h + (cap_ - pos), std::memory_order_release); continue;
            }
            RecordHeader hdr; std::memcpy(&hdr, buf_.get() + pos, sizeof hdr);
            if (hdr.meta == nullptr) {                  // explicit pad
                head_.store(h + (cap_ - pos), std::memory_order_release); continue;
            }
            return buf_.get() + pos;
        }
    }
    void pop(size_t bytes) {
        head_.store(head_.load(std::memory_order_relaxed) + bytes, std::memory_order_release);
    }
    uint64_t droppedCount() const { return dropped_; }

private:
    const size_t                 cap_, mask_;
    std::unique_ptr<std::byte[]> buf_;
    alignas(64) std::atomic<size_t> head_{0};   // consumer writes, producer reads
    alignas(64) std::atomic<size_t> tail_{0};   // producer writes, consumer reads
    alignas(64) size_t cachedHead_{0};          // producer-private
    uint64_t           dropped_{0};             // producer-private
    alignas(64) size_t cachedTail_{0};          // consumer-private
};

// ─────────────────────────────── Logger ───────────────────────────────────
class Logger {
public:
    static Logger& instance() { static Logger l; return l; }

    void addSink(std::unique_ptr<Sink> s) { std::lock_guard g(mu_); sinks_.push_back(std::move(s)); }
    void setLevel(Level l)  { level_.store(l, std::memory_order_relaxed); }
    bool enabled(Level l) const { return l >= level_.load(std::memory_order_relaxed); }

    void start() {
        if (running_.exchange(true)) return;
        consumer_ = std::thread([this] { consumerLoop(); });
    }
    void stop() {
        if (!running_.exchange(false)) return;
        consumer_.join();
    }
    ~Logger() { stop(); }

    // ---- HOT PATH ---------------------------------------------------------
    template <class... Ts>
    void log(const SourceMeta& meta, const Ts&... args) {
        const size_t argBytes = (size_t{0} + ... + argSize(args));
        const size_t total    = sizeof(RecordHeader) + argBytes;
        SpscRing& ring        = localRing();

        size_t reserved;
        std::byte* p = ring.reserve(total, reserved);
        if (!p) return;                                  // dropped, counted
        RecordHeader hdr{now(), &meta, static_cast<uint32_t>(argBytes)};
        std::memcpy(p, &hdr, sizeof hdr);
        p += sizeof hdr;
        ((p = encodeArg(p, args)), ...);
        ring.commit(reserved);
    }

private:
    struct ThreadQueue {
        explicit ThreadQueue(size_t cap) : ring(cap) {}
        SpscRing          ring;
        std::atomic<bool> closed{false};
        uint64_t          reportedDrops{0};
        std::string       tid;
    };
    // RAII: marks queue closed when the owning thread exits; consumer drains, then frees.
    struct ThreadHandle {
        std::shared_ptr<ThreadQueue> q;
        ~ThreadHandle() { if (q) q->closed.store(true, std::memory_order_release); }
    };

    SpscRing& localRing() {
        thread_local ThreadHandle h = [this] {
            auto q = std::make_shared<ThreadQueue>(ringBytes_);
            { std::ostringstream is; is << std::this_thread::get_id(); q->tid = is.str(); }
            std::lock_guard g(mu_);
            queues_.push_back(q);
            return ThreadHandle{q};
        }();
        return h.q->ring;
    }

    static uint64_t now() {
        return static_cast<uint64_t>(
            std::chrono::duration_cast<std::chrono::nanoseconds>(
                std::chrono::system_clock::now().time_since_epoch()).count());
    }

    // ---- COLD PATH (consumer thread) --------------------------------------
    void consumerLoop() {
        std::vector<std::shared_ptr<ThreadQueue>> qs;
        std::string line;
        auto lastFlush = std::chrono::steady_clock::now();

        auto drainOnce = [&]() -> bool {
            { std::lock_guard g(mu_); qs = queues_; }
            bool didWork = false;

            for (;;) {
                // k-way merge: pick the queue whose head record has the smallest timestamp
                ThreadQueue* best = nullptr; const std::byte* bestRec = nullptr; uint64_t bestTs = ~0ull;
                for (auto& q : qs) {
                    if (const std::byte* rec = q->ring.peek()) {
                        RecordHeader hdr; std::memcpy(&hdr, rec, sizeof hdr);
                        if (hdr.ts < bestTs) { bestTs = hdr.ts; best = q.get(); bestRec = rec; }
                    }
                }
                if (!best) break;
                didWork = true;

                RecordHeader hdr; std::memcpy(&hdr, bestRec, sizeof hdr);
                line.clear();
                formatPrefix(line, hdr, *best);
                hdr.meta->format(line, hdr.meta->fmt, bestRec + sizeof hdr);
                best->ring.pop(sizeof hdr + hdr.argBytes);
                dispatch(hdr.meta->level, line);
            }

            // report drops + reap closed queues
            for (auto& q : qs) {
                uint64_t d = q->ring.droppedCount();
                if (d != q->reportedDrops) {
                    line = "[mlog] thread " + q->tid + " dropped " + std::to_string(d - q->reportedDrops) + " records";
                    q->reportedDrops = d;
                    dispatch(Level::Warn, line);
                }
            }
            {
                std::lock_guard g(mu_);
                for (auto it = queues_.begin(); it != queues_.end();)
                    if ((*it)->closed.load(std::memory_order_acquire) && !(*it)->ring.peek()) it = queues_.erase(it);
                    else ++it;
            }
            return didWork;
        };

        while (running_.load(std::memory_order_relaxed)) {
            if (!drainOnce()) std::this_thread::sleep_for(std::chrono::microseconds(50));
            auto t = std::chrono::steady_clock::now();
            if (t - lastFlush > std::chrono::milliseconds(100)) { flushAll(); lastFlush = t; }
        }
        while (drainOnce()) {}   // final drain on shutdown
        flushAll();
    }

    static void formatPrefix(std::string& out, const RecordHeader& hdr, const ThreadQueue& q) {
        using namespace std::chrono;
        const uint64_t ns = hdr.ts % 1'000'000'000ull;
        const time_t   s  = static_cast<time_t>(hdr.ts / 1'000'000'000ull);
        tm t{};
#ifdef _WIN32
        gmtime_s(&t, &s);
#else
        gmtime_r(&s, &t);
#endif
        char buf[40];
        int n = std::snprintf(buf, sizeof buf, "%02d:%02d:%02d.%09llu ",
                              t.tm_hour, t.tm_min, t.tm_sec, static_cast<unsigned long long>(ns));
        out.append(buf, n);
        out.append(levelName(hdr.meta->level)).append(" [").append(q.tid).append("] ");
        // strip directory from file
        const char* f = hdr.meta->file; if (const char* sl = std::strrchr(f, '/')) f = sl + 1;
        out.append(f).push_back(':'); out.append(std::to_string(hdr.meta->line)).append(" | ");
    }

    void dispatch(Level lvl, std::string_view line) {
        // sinks_ only mutated via addSink before/while consumer runs; take the lock cheaply
        std::lock_guard g(mu_);
        for (auto& s : sinks_) if (lvl >= s->minLevel) s->write(lvl, line);
    }
    void flushAll() { std::lock_guard g(mu_); for (auto& s : sinks_) s->flush(); }

    std::mutex                                mu_;
    std::vector<std::unique_ptr<Sink>>        sinks_;
    std::vector<std::shared_ptr<ThreadQueue>> queues_;
    std::atomic<Level>                        level_{Level::Trace};
    std::atomic<bool>                         running_{false};
    std::thread                               consumer_;
    size_t                                    ringBytes_ = 1 << 20;   // 1 MiB per thread
};

}  // namespace mlog

// ────────────────────────────────── Macro ─────────────────────────────────
// The immediately-invoked generic lambda gives us a `static` per callsite whose
// type depends on the argument types, so the format function is resolved once.
#define MLOG(lvl, fmtstr, ...)                                                          \
    do {                                                                                \
        if constexpr ((lvl) >= MLOG_MIN_LEVEL) {                                        \
            auto& mlog_lg_ = ::mlog::Logger::instance();                                \
            if (mlog_lg_.enabled(lvl)) {                                                \
                [&](const auto&... mlog_a_) {                                           \
                    static const ::mlog::SourceMeta mlog_meta_{                         \
                        fmtstr, __FILE__, __LINE__, lvl,                                \
                        &::mlog::formatRecord<std::decay_t<decltype(mlog_a_)>...>};     \
                    mlog_lg_.log(mlog_meta_, mlog_a_...);                               \
                }(__VA_ARGS__);                                                         \
            }                                                                           \
        }                                                                               \
    } while (0)

#define LOG_TRACE(...) MLOG(::mlog::Level::Trace, __VA_ARGS__)
#define LOG_DEBUG(...) MLOG(::mlog::Level::Debug, __VA_ARGS__)
#define LOG_INFO(...)  MLOG(::mlog::Level::Info,  __VA_ARGS__)
#define LOG_WARN(...)  MLOG(::mlog::Level::Warn,  __VA_ARGS__)
#define LOG_ERROR(...) MLOG(::mlog::Level::Error, __VA_ARGS__)
```
