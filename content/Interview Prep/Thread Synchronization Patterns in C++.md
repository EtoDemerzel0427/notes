---
title: "Thread Synchronization Patterns in C++"
slug: "thread-sync-patterns-cpp"
date: "2026-09-21"
tags: ["Interview", "C++", "Concurrency", "Low Latency"]
category: "Interview Prep"
draft: false
---

从 [[Low-Latency Logging for Options Market Making]] 里拆出来的一页。那道题里 mutex + condvar 只是"两个线程怎么交接数据"的**一种**答案。这里把常见模式摆在一起：每种解决什么问题、代价是什么、面试里怎么写。

> 本页所有代码在 Apple Silicon（ARM，弱内存序）上用 `clang++ -std=c++20 -O2` 编译并测试通过，ThreadSanitizer 无报告。在弱序机器上测过比在 x86 上测过更有说服力——x86 的 TSO 会掩盖很多 memory order 的错误（见 §5 的实测）。

---

## 0. 先建立坐标系

线程间"同步"其实是三个不同的问题，很多混乱来自把它们混在一起：

| 问题 | 含义 | 典型工具 |
| --- | --- | --- |
| **互斥（mutual exclusion）** | 同一时刻只有一个线程碰这份数据 | mutex、spinlock |
| **等待/通知（signaling）** | "没活干的时候睡觉，有活了叫醒我" | condvar、semaphore、`atomic::wait`、eventfd |
| **发布（publication）** | 一个线程写好的数据，让另一个线程**安全地看到** | atomic 索引/指针 + memory order、seqlock、RCU |

选型时问四个问题：

1. **几个写者、几个读者？** SPSC / MPSC / MPMC / 单写多读——写者越少，能用的招越便宜。
2. **要传递每一条，还是只要最新值？** 队列 vs. 快照（seqlock）。
3. **等待方可以烧 CPU 吗？** busy-poll vs. park。
4. **谁的延迟重要？** 往往只有一侧是热路径——把成本推给另一侧。

进度保证的术语（面试常问）：

| 术语 | 含义 | 例子 |
| --- | --- | --- |
| blocking | 一个线程被挂起可以让其他线程全部卡住 | mutex |
| lock-free | **系统整体**总有线程在前进；单个线程可能一直重试 | CAS 循环的 MPSC 队列 |
| wait-free | **每个线程**都在有限步内完成 | SPSC ring 的 push/pop |

---

## 1. Mutex + Condition Variable

最通用的组合：mutex 负责互斥，condvar 负责等待/通知。

### 1.1 有界阻塞队列：一把锁、两个条件变量

```cpp
template <class T>
class BoundedBlockingQueue {
public:
    explicit BoundedBlockingQueue(size_t cap) : cap_(cap) {}
    void push(T v) {
        std::unique_lock lk(mu_);
        notFull_.wait(lk, [&] { return q_.size() < cap_ || closed_; });
        if (closed_) return;
        q_.push_back(std::move(v));
        lk.unlock();
        notEmpty_.notify_one();
    }
    bool tryPush(T v) {                       // 非阻塞版本 == "满了就丢"策略
        {
            std::lock_guard g(mu_);
            if (q_.size() >= cap_ || closed_) return false;
            q_.push_back(std::move(v));
        }
        notEmpty_.notify_one();
        return true;
    }
    std::optional<T> pop() {                  // nullopt == 已关闭且已排空
        std::unique_lock lk(mu_);
        notEmpty_.wait(lk, [&] { return !q_.empty() || closed_; });
        if (q_.empty()) return std::nullopt;
        T v = std::move(q_.front());
        q_.pop_front();
        lk.unlock();
        notFull_.notify_one();
        return v;
    }
    void close() {
        { std::lock_guard g(mu_); closed_ = true; }
        notEmpty_.notify_all();
        notFull_.notify_all();
    }
private:
    const size_t cap_;
    std::mutex mu_;
    std::condition_variable notEmpty_, notFull_;
    std::deque<T> q_;
    bool closed_ = false;
};
```

`push`（阻塞）和 `tryPush`（丢弃）就是 logging 那题里 backpressure 策略的两端——**同一个数据结构，策略只是接口的选择**。

### 1.2 condvar 的五条铁律

| 规则 | 原因 |
| --- | --- |
| `wait` 永远带谓词 | **Spurious wakeup**：标准允许无故醒来。**Lost wakeup**：`notify` 发生在 `wait` 之前就丢了。谓词版本等价于 `while (!pred()) wait(lk);`，两者都防住 |
| 修改谓词依赖的状态时**必须持锁** | 哪怕那个状态是 atomic。否则：等待者检查谓词（false）→ 你改状态 + notify → 等待者才进入 wait → 永远睡下去 |
| `notify` 放在解锁之后 | 否则被唤醒的线程立刻撞上你还没放的锁（"hurry up and wait"）。在锁内 notify 也是正确的，只是多一次无谓的切换 |
| 两个条件用两个 condvar | 只用一个的话，`notify_one` 可能叫醒"错的那一类"等待者（生产者叫醒了另一个生产者）→ 只能用 `notify_all` → thundering herd |
| 关闭时用 `notify_all` | 所有等待者都需要醒来看到 `closed_` |

### 1.3 Double buffering（swap 技巧）

消费者一次把整个队列 `swap` 走，临界区缩到一次指针交换，处理全在锁外：

```cpp
std::deque<T> local;
{
    std::unique_lock lk(mu_);
    cv_.wait(lk, [&] { return stop_ || !q_.empty(); });
    local.swap(q_);
}
for (auto& x : local) process(x);
```

适用于单消费者。这是 mutex 方案里**性价比最高的一个优化**，面试里写 mutex 队列时应该默认这么写。

### 1.4 什么时候 mutex 就是正确答案

- 临界区短、争用低 → 无争用的 `lock`/`unlock` 是一次用户态 CAS，约 20ns，**不进内核**。
- 需要保护的是一个复杂不变式（多个字段一起变）。
- 不在延迟关键路径上：注册、配置、启停。logging 那题里 `mu_` 保护 registry 就是这种用法。

> "有锁 = 慢"是误解。慢的是**争用**和**持锁时被调度走**。先问"这把锁会被争用吗、在热路径上吗"，再决定要不要换掉它。

---

## 2. Semaphore（C++20）

condvar 等的是"**谓词**成立"，semaphore 等的是"**计数** > 0"。当你等待的东西天然是可数资源时，semaphore 更直接，而且**没有 lost wakeup 问题**——`release` 先于 `acquire` 发生也不会丢，计数会记住。

```cpp
template <class T, size_t N>
class SemQueue {
public:
    void push(T v) {
        slots_.acquire();                                   // 等一个空槽
        { std::lock_guard g(mu_); buf_[tail_++ % N] = std::move(v); }
        items_.release();
    }
    T pop() {
        items_.acquire();                                   // 等一个元素
        T v;
        { std::lock_guard g(mu_); v = std::move(buf_[head_++ % N]); }
        slots_.release();
        return v;
    }
private:
    std::counting_semaphore<N> slots_{N}, items_{0};
    std::mutex mu_;
    T buf_[N];
    size_t head_ = 0, tail_ = 0;
};
```

- 这是操作系统教科书里的经典 producer-consumer 解法：两个 semaphore 管"满/空"，一把 mutex 管缓冲区本身。
- SPSC 场景下 `mu_` 可以去掉（head/tail 各自只有一个写者）。
- `std::binary_semaphore` 可以当"一次性事件"用；和 mutex 的区别是**没有所有权**，A 线程 acquire、B 线程 release 是合法的。
- `try_acquire()` / `try_acquire_for()` 提供非阻塞与超时版本。

---

## 3. Spinlock

```cpp
class SpinLock {
public:
    void lock() {
        for (;;) {
            if (!locked_.exchange(true, std::memory_order_acquire)) return;
            while (locked_.load(std::memory_order_relaxed)) cpuRelax();   // 在"读"上自旋，而不是在 xchg 上
        }
    }
    void unlock() { locked_.store(false, std::memory_order_release); }
private:
    static void cpuRelax() {
#if defined(__x86_64__)
        __builtin_ia32_pause();
#elif defined(__aarch64__)
        __asm__ volatile("yield");
#endif
    }
    std::atomic<bool> locked_{false};
};
```

- **Test-and-test-and-set**：朴素写法是 `while (locked_.exchange(true));`——每次循环都是一次写，cache line 在所有等待者之间以 Modified 状态来回弹。先用 `load` 自旋（line 保持 Shared），看到空闲了再 `exchange`。
- **`pause`**：告诉 CPU 这是自旋等待——避免退出循环时的 memory-order misspeculation 惩罚、省电、把执行资源让给同核的超线程。
- **什么时候用**：临界区只有几十纳秒、且持锁线程**不会被调度走**（线程 pin 核、核数 ≥ 线程数）。
- **什么时候是灾难**：持锁者被抢占 → 所有等待者空烧整个时间片。用户态没法关抢占，所以通用代码里 spinlock 几乎总是比 `std::mutex` 差——`std::mutex` 自己就是"先自旋一小会儿再 futex 睡眠"的混合体。
- 满足 *Lockable* 要求（`lock`/`unlock`/`try_lock`）就能配合 `std::lock_guard` 使用。

---

## 4. `std::atomic::wait` / `notify`（C++20）

标准库版的 **futex**："如果这个 atomic 的值还等于 X，就让我睡；有人 notify 时叫醒我。"

```cpp
class Event {
public:
    void signal() { seq_.fetch_add(1, std::memory_order_release); seq_.notify_one(); }
    void waitChange(uint32_t seen) { seq_.wait(seen, std::memory_order_acquire); }  // 值 != seen 时返回
    uint32_t current() const { return seq_.load(std::memory_order_acquire); }
private:
    std::atomic<uint32_t> seq_{0};
};
```

- 相比 condvar：不需要 mutex、状态就是 atomic 本身、"检查值 + 入睡"由内核原子地完成 → 没有 lost wakeup 窗口。
- 用法模式：消费者 `seen = current(); 检查队列; 队列为空则 waitChange(seen);`。先读 seq 再检查队列，保证检查之后到入睡之前的 signal 不会丢。
- `notify_one` 在没有等待者时，好的实现只是一次用户态检查；有等待者时是 futex wake syscall——**和 condvar 一样，生产者要为唤醒付费**。
- 这是"无锁队列 + 可睡眠的消费者"的标准拼法：数据走 lock-free ring，等待走 `atomic::wait`。

---

## 5. Seqlock：单写者、只要最新值

场景：行情线程不断更新某个 quote，多个策略线程读**最新值**。不需要每一条，只要读到的是一份**自洽**的快照；而且**写者绝不能被读者阻塞**。

```cpp
struct Quote { int64_t bid, ask; };

class QuoteSeqLock {
public:
    void store(Quote q) {                                       // 单写者
        const uint32_t s = seq_.load(std::memory_order_relaxed);
        seq_.store(s + 1);                                      // 奇数 = 正在写
        bid_.store(q.bid, std::memory_order_relaxed);
        ask_.store(q.ask, std::memory_order_relaxed);
        seq_.store(s + 2);                                      // 偶数 = 稳定
    }
    Quote load() const {
        Quote q; uint32_t s0, s1;
        do {
            s0    = seq_.load();
            q.bid = bid_.load();                                // 不能是 relaxed（见下）
            q.ask = ask_.load();
            s1    = seq_.load();
        } while (s0 != s1 || (s0 & 1));
        return q;
    }
private:
    std::atomic<uint32_t> seq_{0};
    std::atomic<int64_t>  bid_{0}, ask_{0};
};
```

- 写者 wait-free，永远不等任何人；读者乐观读取，校验失败就重试（读者可能被饿死，但写很快时实际不会）。
- 读者**不写任何共享内存** → 读者之间、读者与写者之间没有 cache line 争用。这是它比 `shared_mutex` 好得多的地方（后者的读锁也要写计数器）。
- 数据必须 trivially copyable，且读到"撕裂"的中间值时不能崩（不能是指针然后去解引用）。

### ⚠️ 实测：memory order 写错在这里会**真的出错**

我第一版把数据读取写成了 `relaxed`，在 ARM 上跑 2000 万次写，结果：

| 变体 | 撕裂读次数 |
| --- | --- |
| 数据是普通 struct，seq 用 `release`/`acquire` | **94,267** |
| 数据是 atomic 字段，数据 **load 用 `relaxed`**，seq 用 `seq_cst` | **11,595** |
| 同上，但 `-O0` | 0 |
| 数据 load 用默认 `seq_cst`（上面的代码） | 0 |

原因：`relaxed` 不只是给硬件的许可，**也是给编译器的许可**。读循环里 `bid`/`ask` 的值只在循环退出后才用到，编译器于是合法地把这两次 load **挪到了 `s1` 校验之后**——校验通过了，然后才去读数据，保护形同虚设。`-O0` 不出错、两种 load 的汇编都是同一条 `ldr`，证明这是编译器重排而不是硬件乱序。

教训：
- 写对 memory order 的意思是"声明我依赖哪种顺序保证"；声明少了，优化器就会用掉你没声明的自由度。
- 拿不准就用默认的 `seq_cst`。x86 上 `seq_cst` 的 **load** 本来就是普通 `mov`，没有任何额外成本；贵的只是 `seq_cst` 的 **store**（`xchg`）。
- 生产实现（Linux 内核、各种开源 Seqlock）通常用普通 struct + `memcpy` + 编译器屏障（`barrier()` / `noinline`）。这在 C++ 标准下形式上是 data race（UB），靠的是对具体编译器行为的掌控。面试里说出"形式上是 UB，实践中怎么控制"就是加分项。

---

## 6. 读多写少的配置：Copy-on-Write 快照（RCU 风格）

场景：sink 列表、log level 配置、路由表——热路径频繁读，极少改。

```cpp
class ConfigHolder {
public:
    std::shared_ptr<const Config> get() const { return std::atomic_load(&cur_); }
    void update(Config c) {
        std::atomic_store(&cur_, std::shared_ptr<const Config>(std::make_shared<Config>(std::move(c))));
    }
private:
    std::shared_ptr<const Config> cur_ = std::make_shared<Config>();
};
```

- 写者：拷贝一份 → 修改 → 原子地替换指针。旧版本在最后一个读者释放 `shared_ptr` 时自动回收——引用计数替你解决了"什么时候能安全 delete"这个 RCU 里最难的问题。
- C++20 的正式写法是 `std::atomic<std::shared_ptr<T>>`（上面的自由函数在 C++20 被弃用；但 libc++ 目前还没实现新写法）。
- **注意**：`atomic<shared_ptr>` 在主流实现里**不是 lock-free 的**（内部是一个小自旋锁），而且每次 `get()` 都有引用计数的原子增减 → 共享 cache line 的写。真正的热路径上应该：每个线程缓存一份快照 + 一个 atomic 版本号，只有版本号变了才重新 `get()`。
- 这正是 logging 那题里"`dispatch` 每行都拿 `mu_` 保护 `sinks_`"的正解之一。

---

## 7. Lock-free 队列家族

| 类型 | 热路径 | 难点 | 面试里 |
| --- | --- | --- | --- |
| **SPSC ring** | 一次 plain store，wait-free | memory order、false sharing、index cache | **要能手写**。完整推导见 [[Low-Latency Logging for Options Market Making]] §6 |
| **MPSC**（多生产者） | 一次 `exchange` 或 CAS 循环 | 生产者之间争用同一条 cache line | 讲清思路即可 |
| **MPMC** | 两端都 CAS | ABA、内存回收 | 讲清为什么难 |
| **Disruptor** | CAS claim 一个序号 → 写槽 → publish | 消费者要处理"已 claim 未 publish"的空洞 | 知道思想 |

### 7.1 CAS 循环的基本形态

```cpp
// Treiber stack 的 push：最小的 lock-free 结构
void push(Node* n) {
    n->next = head.load(std::memory_order_relaxed);
    while (!head.compare_exchange_weak(n->next, n,
                                       std::memory_order_release,
                                       std::memory_order_relaxed)) {}
}
```

- `compare_exchange_weak` 失败时会把**当前值**写回第一个参数，所以循环体可以是空的。
- `weak` 允许伪失败（ARM 的 LL/SC 被中断打断时），放在循环里用它；不在循环里用 `strong`。
- 这是 lock-free 而不是 wait-free：倒霉的线程可以一直输掉 CAS。

### 7.2 ABA 问题

线程 1 读到 `head == A`，准备 CAS(A → A.next)；被挂起。线程 2 pop A、pop B、又 push 回 A（同一地址）。线程 1 醒来，CAS 成功——但它手里的 `A.next` 还是已经被释放的 B。

对策：带版本号的指针（tagged pointer / 双字 CAS）、hazard pointers、epoch-based reclamation，或者**干脆不释放节点**（预分配池 + 索引代替指针）。

> 为什么 SPSC ring 没有这些问题：没有 CAS（每个索引单写者）、没有动态节点（预分配数组）、索引单调递增（不会"变回去"）。**能把问题约束成 SPSC，就别去碰 MPMC**——这是 logging 那题选 per-thread SPSC 的深层原因。

### 7.3 Vyukov MPSC 队列（知道思想）

生产者：`prev = tail.exchange(node); prev->next = node;`——一次 `xchg` 完成入队，wait-free。代价是这两步之间有一个极短的窗口，消费者会看到链表"断开"（`next == nullptr` 但 `tail` 已经前移），需要自旋等一下。是 actor 系统邮箱的经典实现。

---

## 8. "无锁队列 + 会睡觉的消费者"：Dekker 陷阱

想省 CPU，让消费者在没活时睡觉，又不想让生产者每次 push 都付一次 notify 的 syscall。直觉方案：

```text
消费者：                              生产者：
  sleeping = true                       push(item)
  if (queue.empty()) park()             if (sleeping) wake()
  sleeping = false
```

这是一个 **store-load 模式**：双方都是"先写自己的标志，再读对方的状态"。如果双方都读到了旧值——消费者没看到新元素、生产者没看到 `sleeping == true`——消费者就带着一个非空队列睡死了。

- 在 x86 上，store buffer 会让这种情况真实发生。**必须让这两个 store 是 `seq_cst`**（编译成 `xchg`，刷掉 store buffer），这是 `seq_cst` 少数不可替代的场合。
- 代价：生产者热路径上多了一次 `xchg`（~20 cycles）+ 一次对 `sleeping` 的读。比 syscall 便宜得多，但不是零。
- 保险做法：消费者 park 时带超时（比如 1ms），把"睡死"降级为"偶尔晚 1ms"。
- 做市系统的常见选择是根本不睡：消费者 busy-poll，独占一个非关键核。**用一个核换掉整类 bug。**

---

## 9. 优雅停机：`std::jthread` + `stop_token`（C++20）

```cpp
class Worker {
public:
    Worker() : th_([this](std::stop_token st) { run(st); }) {}
    void post(int v) { { std::lock_guard g(mu_); q_.push_back(v); } cv_.notify_one(); }
private:
    void run(std::stop_token st) {
        std::unique_lock lk(mu_);
        for (;;) {
            cv_.wait(lk, st, [&] { return !q_.empty(); });      // notify 或 stop 请求都会唤醒
            if (q_.empty()) return;                             // 被要求停止且已排空
            while (!q_.empty()) { handle(q_.front()); q_.pop_front(); }
        }
    }
    std::mutex mu_;
    std::condition_variable_any cv_;                            // 注意是 _any
    std::deque<int> q_;
    std::jthread th_;                                           // 最后一个成员
};
```

- `jthread` 析构时自动 `request_stop()` + `join()`——不会再因为忘了 join 而 `std::terminate`。
- `condition_variable_any::wait(lk, stop_token, pred)`：stop 请求会唤醒等待者，省掉手写的 `stop_` 标志和析构函数里那套"加锁、置位、notify、join"。
- **成员声明顺序**：`th_` 放最后。成员按声明顺序构造（线程启动时 `mu_`/`cv_` 已存在），按逆序析构（线程先 join，然后 `mu_`/`cv_` 才销毁）。顺序写反是一个真实且隐蔽的 bug。

---

## 10. 其它值得知道的

- **`std::shared_mutex`（读写锁）**：听上去适合读多写少，实际上读锁也要原子地改共享计数器 → 读者之间照样有 cache line 争用。临界区很短时，它经常比普通 `mutex` 还慢。读多写少的正解通常是 §5 或 §6。
- **`std::call_once` / function-local static**：一次性初始化。后者由编译器保证线程安全，代价是每次经过检查一个 guard。
- **`std::scoped_lock(m1, m2)`**：一次锁多把，内部用避免死锁的算法。需要同时持有两把锁时永远用它，而不是手写两次 `lock_guard`。
- **Thread-per-core / share-nothing**：终极方案是不共享。每个核一个线程、各自拥有数据，线程间只通过 SPSC 队列传消息。做市系统的典型架构（行情线程 → 策略线程 → 订单线程，pipeline 的每一段之间是一条 SPSC ring），也是 Seastar/ScyllaDB 的设计哲学。

---

## 11. 选型速查

| 场景 | 首选 | 理由 |
| --- | --- | --- |
| 通用任务队列，不在延迟关键路径上 | mutex + condvar + swap | 简单、正确、好维护 |
| 等待的是可数资源 | `counting_semaphore` | 没有 lost wakeup，没有谓词 |
| 热路径 → 后台线程，1 对 1 | **SPSC ring**，消费者 busy-poll | wait-free，生产者一次 plain store |
| 多个热路径线程 → 一个后台线程 | **每线程一个 SPSC ring** | 把争用变成消费者的轮询 |
| 只要最新值，单写多读 | **seqlock** | 写者不被任何人阻塞，读者不写共享内存 |
| 读多写少的配置 | COW 快照 + 版本号 | 读路径近乎免费 |
| 极短临界区 + 线程全部 pin 核 | spinlock | 省掉 futex；持锁者不会被调度走是前提 |
| 无锁队列但消费者要省 CPU | ring + `atomic::wait`，或 spin-then-park | 注意 §8 的 Dekker 陷阱 |
| 线程生命周期管理 | `jthread` + `stop_token` | 自动 join，可中断的等待 |

**面试时的总原则**：先给出最简单的正确方案（通常是 mutex + condvar），说清它的成本在哪，再根据题目的延迟/吞吐约束逐步替换。每一步都说出"我在消除哪一项成本、换来了什么新的约束"。
