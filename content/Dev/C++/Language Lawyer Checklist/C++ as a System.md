---
title: C++ Core Language
slug: cpp-core-language
date: 2026-01-10
tags: [C++, Overview]
category: Dev
draft: false
---

## 学习目标
Core Language 定义了 C++ 抽象机器（Abstract Machine）的物理法则。本模块涵盖从微观执行模型到宏观元编程的所有核心规则。
1.  **Execution & Behavior**: 抽象机器、未定义行为 (UB)、严格别名规则。
2.  **Object Model**: 对象生命周期、值类别、内存布局。
3.  **Lookup & Semantics**: 名称解析、初始化、重载决议。
4.  **Templates & Metaprogramming**: 类型推导、SFINAE/Concepts、编译期计算。
5.  **Concurrency Model**: 内存序、Happens-before 关系、数据竞争。

## Part 0. ISO C++ Standard Working Draft
1. 在线阅读版：https://eel.is/c++draft/ 对应Github仓库的最新提交。
2. 资源汇总：https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/ C++ 23: `N4950`, C++ 26: `N5032`.

## Part 1: 执行模型与行为 (Execution Model & Behavior)
**对应标准文档**：`[intro.execution]`, `[intro.behavior]`, `[basic.lval]`

* 1.1 行为分类 (Classes of Behavior)
  * Undefined Behavior (UB)
    * **本质**：编译器优化的契约。编译器假设 UB 永远不会发生，从而消除冗余代码（如删除溢出检查）。
    * **常见触发**：有符号整数溢出、空指针解引用、越界访问、在生命周期结束后的对象访问。
  * Unspecified vs Implementation-defined
    * **Unspecified**: 行为不确定，且无需文档化（如函数参数求值顺序）。
    * **Implementation-defined**: 行为不确定，但必须文档化（如 `int` 的字节宽）。

* 1.2 访问与别名 (Access & Aliasing)
  * Strict Aliasing Rule (严格别名规则)
    * **法则**：不能通过不同类型的指针（`char*` 等例外）访问同一个对象。
    * **后果**：违反此规则会导致编译器进行错误的指令重排（假设两个指针不相关）。
  * Sequenced-before 关系
    * 定义单线程内的求值偏序。解决 `i = i++` 为何是 UB 的理论依据。

## Part 2: 对象模型 (The Object Model)
**对应标准文档**：`[basic.types]`, `[basic.life]`, `[dcl.init]`

### 2.1 值类别 (Value Categories)
* **体系**：`glvalue` (身份), `prvalue` (计算/初始化), `xvalue` (即将销毁/移动)。
* **Temporary Materialization (临时对象实质化)**
    * C++17 引入：`prvalue` 在需要地址绑定时才转换为 `xvalue`（临时对象），在此之前不占用存储。

### 2.2 对象生命周期 (Lifetime)
* **Storage Duration**: Static, Thread, Automatic, Dynamic。
* **Construction/Destruction**:
     * **Triviality (平凡性)**：什么是 Trivial, Standard-layout, POD？为什么 memcpy 只能用于 trivial types？
     * **Construction/Destruction 顺序**：包含虚继承、数组成员、全局对象时的精确构造析构顺序。
    * 全局对象初始化顺序的不确定性 (Static Initialization Order Fiasco)。
* **Pointer Provenance (指针溯源)** (C++20/26)
    * 指针不仅是地址值，还携带“来源”信息。即使两个指针地址位相同，若来源不同，互换使用可能导致 UB。

*  **与现在工作（Trading）的 关联**：理解 Memory Layout 和 Padding 是做低延迟系统（避免 False Sharing，优化 Cache Line）的基础。
* **参考书：**《Inside the C++ Object Model》（虽然老，但关于 vptr/vtable 和对象布局的原理依然是基石）。



## Part 3: 语义分析与名称查找 (Semantics & Name Lookup)
> 对应标准章节：`[basic.lookup]`, `[over]`, `[class]`\
> 核心逻辑：编译器看到一个名字（Name），它是如何找到对应实体（Entity）的？

* 3.1 名称查找 (Name Lookup)
  * 系统化知识点：
     * **Unqualified Lookup** vs **Qualified Lookup**。
     * **ADL (Argument-Dependent Lookup)**：为什么 `std::swap` 有时不需要写 `std::`？ADL 的触发条件和陷阱（比如两阶段查找）。
    *  **Two-phase Lookup**: 模板中的依赖名（Dependent Names）查找延迟到实例化阶段，而非定义阶段。
    * **Injected-class-name**：类名在类内部的特殊地位。

  * 3.2 重载决议 (Overload Resolution)
    * 系统化知识点：
      * 候选函数集（Candidate set）是如何构建的？
      * 可行函数（Viable functions）的筛选标准。
      * **Ranking**：Exact Match > Promotion > Conversion。
      * **Tie-breakers**: 非模板优先于模板；更受限的 Concept (C++20) 优先于通用 Concept

  * 3.3 类型转换 (Conversions)
    * 系统化知识点：
      * **Implicit Conversion**s：Standard conversion sequence vs User-defined conversion sequence。
      * `explicit` 关键字在 `bool` 上下文中的特殊行为（Contextual conversion to bool）。
  * 3.4 初始化 (Initialization)
    * **分类**: Default, Value, Direct, Copy, List, Aggregate。
    * **Most Vexing Parse**: 解析歧义（`T x()` 被解析为函数声明）。
    * **Narrowing Conversion**: 列表初始化 `{}` 禁止窄化转换（如 `double` -> `int`）。

## Part 4: 模板与元编程 (Templates & Metaprogramming)
> 对应标准章节：`[temp]` 。

* 4.1 模板推导与实例化 (Deduction & Instantiation)
  * 系统化知识点：
    * **Type Deduction**: 引用折叠 (Reference Collapsing) 与万能引用 (Forwarding References)。
    * **CTAD (Class Template Argument Deduction)**: C++17 允许类模板自动推导参数 (`vector v = {1};`)。
    * **POI (Point of Instantiation)**: 实例化发生的位置，决定了哪些名字可见。
    * **SFINAE (Substitution Failure Is Not An Error)**: 替换失败仅导致从重载集中剔除，而非编译错误。

* 4.2 特化与偏特化 (Specialization & Partial Specialization)
  * 系统化知识点：
    * 函数模板不能偏特化（只能重载），类模板可以偏特化。为什么？
    * **SFINAE**：std::enable_if 的底层原理（替换失败不是错误）。

* 4.3 概念与约束 (Concepts & Constraints) (C++20)
  * 系统化知识点：
    * Requires-clause, Atomic constraints, Constraint normalization。
    * **Concepts**：替代 `enable_if` 的现代化约束机制。
    * **Constraint Normalization & Subsumption**： 编译器如何分解约束表达式（Atomic Constraints）并判定“包含关系”，以决定哪个模板更特化（More Specialized）。
* 4.3 编译期计算 (Compile-time Evaluation)
    * **constexpr**: 可在编译期求值的函数/变量。
    * **consteval (C++20)**: **强制**在编译期求值的即时函数 (Immediate   Function)。
    * **if consteval (C++23)**: 检测当前是否处于常量求值上下文（用于编写 Runtime/Compile-time 双路径优化的代码）。


## Part 5: 并发内存模型 (Concurrency Memory Model)
**对应标准文档**：`[intro.multithread]`, `[atomics.order]`

* 5.1 核心关系 (Ordering Relations)
  * **Happens-before**: 跨线程可见性的基础偏序关系。
  * **Synchronizes-with**: 原子操作之间的同步点（如 Release 操作与 Acquire 操作建立同步）。
  * **Dependency-ordered-before**: `memory_order_consume` (极少使用，但在标准中存在) 建立的数据依赖顺序。

*  5.2 内存序语义 (Memory Order Semantics)
   * **Relaxed (`memory_order_relaxed`)**: 仅保证原子性，不保证顺序（无同步）。
   * **Acquire/Release (`memory_order_acquire`, `memory_order_release`)**: 建立 Critical Section 的标准方式，防止读写指令越过屏障。
   * **Sequentially Consistent (`memory_order_seq_cst`)**: 全局全序（最强保证，性能开销最大）。

