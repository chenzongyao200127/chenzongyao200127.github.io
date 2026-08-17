> 原文链接：https://abseil.io/fast/hints.html#performance-hints
> 姊妹篇：[[「译」Performance Hints]]（C++ 原味版）。本篇把原文中所有 C++ 类型与 CL 示例整体换成 Rust 等价物，供不熟悉 C++ 的读者独立阅读。一切技术判断以原文为准，Rust 映射为译者添加。

## 关于这一版

原文的通用原则（估算、测量、算法改进、缓存友好、批量摊销、快速路径）与语言无关，真正“C++ 味”的是那些具体类型（`std::string_view`、`absl::InlinedVector`、`absl::flat_hash_map`、`StatusOr` 等）和 Google 内部库抽象。本篇保留原文的章节结构与散文论述，只把这些具体载体换成 Rust 世界里的对应物，并在有意思的地方点出 Rust 与 C++ 在这条建议上的差异——很多在 C++ 里需要“刻意为之”的优化，在 Rust 里恰恰是默认行为（比如移动语义、独占可变借用），这本身就是理解“机械同理心”的好切口。

代码片段追求“读得懂 + 说清动因”，不追求可直接 `cargo build`；涉及的第三方 crate 我会标出名字，方便你去 docs.rs 查。

### C++ → Rust 概念映射总表

| 原文 C++ 载体 | Rust 等价物 | 说明 / 动因 |
| --- | --- | --- |
| `std::string_view` | `&str` | 借用而不拥有，零拷贝视图 |
| `absl::Span<T>` / `std::span<T>` | `&[T]` / `&mut [T]`（切片） | 语言内建，调用者自选底层容器 |
| `absl::FunctionRef<R(Args…)>` | `&dyn Fn(Args) -> R` 或 `impl Fn(…)` | 不拥有闭包，避免装箱分配 |
| `std::vector<T>` | `Vec<T>` | 堆上连续存储 |
| `absl::InlinedVector<T, N>` | `smallvec::SmallVec<[T; N]>` / `arrayvec::ArrayVec<T, N>` | 小容量内联栈上，省堆分配 |
| `absl::flat_hash_map/set`（Swiss Tables） | `std::collections::HashMap/HashSet` | std 自 1.36 起底层就是 hashbrown（Swiss Tables 的 Rust 移植），SIMD 探测已内建 |
| `absl::btree_map/set` | `std::collections::BTreeMap/BTreeSet` | 缓存友好的有序容器 |
| `std::map` / `std::unordered_map` | `BTreeMap` / `HashMap` | — |
| `std::vector<bool>` / `InlinedBitVector` | `bitvec`、`fixedbitset`、`bit-set` crate | 位打包，集合运算变位运算 |
| `absl::FixedArray<T>` | `Box<[T]>` / `Vec<T>`（`with_capacity` 后不再增长） | 定长堆数组 |
| Arena（`google::protobuf::Arena` 等） | `bumpalo::Bump`、`typed-arena`、`id-arena`、`slotmap` | 批量分配、批量释放、免逐个析构 |
| 用索引代替指针 | `Vec<T>` + `u32` 索引 / `id-arena` / `slotmap` | Rust 里构图的惯用法，同时绕开借用检查器的自引用难题 |
| `absl::Status` / `StatusOr<T>` | `Result<(), E>` / `Result<T, E>` | 热路径上尽量返回裸值或 `Option`，别无脑套 `Result` |
| move vs copy | Rust 默认移动；`Copy` 标记廉价按位复制；`.clone()` 显式深拷贝 | “优先 move”在 Rust 里是编译器默认，拷贝必须显式写出 |
| `reserve` / `resize` | `Vec::with_capacity` / `Vec::reserve` | — |
| `std::sort` vs `std::stable_sort` | `sort_unstable` vs `sort` | 不需要稳定性就用 unstable，省内部缓冲 |
| `StrCat` / `StringPrintf` | `format!` / `write!` / `String::push_str` | 热路径避免 `format!`，直接拼 buffer |
| `alignas(64)` 防伪共享 | `#[repr(align(64))]`、`crossbeam_utils::CachePadded<T>` | 把互相独立的可变字段隔到不同缓存行 |
| lock-free map | `dashmap`、`arc-swap`、`crossbeam` | 分片/无锁并发 |
| 线程兼容 vs 线程安全 | `Send`/`Sync` + 外部 `Mutex`/`RwLock` vs 内部同步 | 借用检查器让“线程兼容”成为零成本默认 |
| protobuf | `prost` / `protobuf` crate；`bytes::Bytes` 做零拷贝 | Rust 里 arena 关联性弱，零拷贝靠 `Bytes` |
| pprof | `cargo flamegraph`、`samply`、`pprof-rs`、`perf` | Rust 生态的采样剖析工具 |
| Lazy init | `std::sync::LazyLock`/`OnceLock`（1.80+）、`once_cell` | 首次使用时才初始化 |
| 手动展开 / 批量字节 | 迭代器 + `chunks_exact(N)`、`std::simd`（nightly）、`bytemuck` | 让自动向量化生效 |
| `#[inline]` 控制 | `#[inline]` / `#[inline(always)]` / `#[inline(never)]` / `#[cold]` | 慢路径标 `#[cold]` 拆出内联 |

---

（下文“思考性能的重要性”“估算”“测量”三节与语言无关，忠实沿用原文论述；从“API 设计考量”起开始出现 Rust 化的示例。）

# 性能优化建议 (Performance Hints) · Rust 版

> 作者：Jeff Dean, Sanjay Ghemawat
> 原始版本：2023/07/27，最后更新：2025/12/16

多年来，我们（Jeff 和 Sanjay）在各种代码片段的性能调优上投入了大量精力。自 Google 成立之初，提升软件性能就至关重要，因为这让我们能够为更多用户提供更多服务。我们编写本文档旨在总结在做此类工作时运用的一些通用原则和特定技巧，并尝试挑选有代表性的源代码变更（Change Lists，简称 CLs）作为示例。原文的具体建议大多涉及 C++ 类型与 CL，但其通用原则同样适用于其他语言——本篇即把它们落到 Rust 上。

本文档聚焦于单个二进制文件上下文中的通用性能调优，不涵盖分布式系统或机器学习（ML）硬件的性能调优（这些本身就是庞大的领域）。

## 思考性能的重要性

Knuth（高德纳）常被断章取义地引用为“过早优化是万恶之源”。完整版本是：“在 97% 的情况下，我们要忽略微小的效率提升：过早优化是万恶之源。然而，我们绝不能放弃那关键的 3% 的机会。” 本文档正是关于这关键的 3%。Knuth 还有一段更具说服力的引言：

> “从示例 2 到示例 2a 的速度提升仅约 12%，许多人可能会认为这微不足道。当今许多软件工程师的传统观念是忽略细微处的效率；但我认为，这仅仅是对那些实行‘省小钱亏大钱’、无法调试或维护其‘优化后’程序的程序员所犯滥用行为的过度反应。
>
> 在成熟的工程学科中，轻易获得 12% 的改进绝不会被视为微不足道；我相信这一观点同样应在软件工程中占据主导地位。当然，我不会在一次性任务上费心做这种优化，但当问题涉及编写高质量程序时，我不想局限于那些剥夺我获得这种效率的工具。”

许多人会说：“让我们先用尽可能简单的方式编写代码，等以后能做性能剖析（Profiling）时再处理性能问题。” 然而这种方法通常是错误的：

1. 如果在开发大型系统时无视所有性能问题，最终你会得到一个扁平的性能概况（Flat Profile），即没有明显热点，因为损耗遍布各处，很难判断从何着手。
2. 如果你在开发一个供他人使用的库，遇到性能问题的往往是那些无法轻易改进它的人（他们得去理解别的团队的代码细节，还要协商优化的优先级）。
3. 当系统处于高负载使用状态时，进行重大更改会更加困难。
4. 很难判断性能问题是否易于解决，于是最终可能采用昂贵的兜底方案，如过度复制或严重超配服务资源。

相反，我们建议：**在编写代码时，如果对可读性或复杂度没有显著影响，请尽量选择更快的替代方案。**

## 估算 (Estimation)

如果你能对正在编写的代码的性能重要性建立直觉，你就能做出更明智的决策（例如，以性能为名引入多少额外复杂度是合理的）。

编写代码时估算性能的一些技巧：

* **是测试代码吗？** 如果是，主要关注算法和数据结构的渐进复杂度。（注：开发周期耗时也重要，避免编写运行过久的测试。）
* **是特定于应用的代码吗？** 试着弄清这段代码的性能有多重要。通常只需区分它是初始化/设置代码，还是位于热点路径（Hot Paths）上的代码（例如服务中处理每个请求的代码）。
* **是将被许多应用使用的库代码吗？** 这种情况很难判断它会变得多敏感，因此遵循本文的简单技巧尤为重要。例如，若要存储一个通常元素很少的向量，用 `SmallVec<[T; N]>`（`smallvec` crate）而不是 `Vec<T>`。遵循这类技巧并不难，也不会给系统增加非局部的复杂度；而一旦这段代码最终确实很热，它从一开始就具备较高性能，剖析时也更容易找到下一个关注点。

在选择性能特征不同的方案时，可以靠**封底计算（Back-of-the-envelope calculations）**做稍深的分析：快速给出各方案非常粗略的估算，据此舍弃某些方案而无需真正实现它们。

估算这样进行：

1. 估算各类底层操作的数量，例如磁盘寻道次数、网络往返次数、传输字节数等。
2. 将每种昂贵操作的数量乘以其粗略成本，再相加。
3. 上述给出的是**资源使用成本**。如果你关心**延迟**且系统存在并发，部分成本会重叠，需要稍复杂的分析来估算延迟。

下表列出一些底层操作的粗略成本（2007 年斯坦福演讲表格的更新版）：

| 操作类型 (Operation)              | 粗略耗时 (Rough Cost) |
| ----------------------------- | ----------------- |
| L1 缓存引用 (L1 cache reference)  | 0.5 ns            |
| L2 缓存引用 (L2 cache reference)  | 3 ns              |
| 分支预测错误 (Branch mispredict)    | 5 ns              |
| 互斥锁 加锁/解锁 (无竞争)               | 15 ns             |
| 主内存引用 (Main memory reference) | 50 ns             |
| 使用 Snappy 压缩 1K 字节            | 1,000 ns          |
| 从 SSD 读取 4KB                  | 20,000 ns         |
| 同一数据中心内的网络往返                  | 50,000 ns         |
| 从内存顺序读取 1MB                   | 64,000 ns         |
| 在 100 Gbps 网络上读取 1MB          | 100,000 ns        |
| 从 SSD 读取 1MB                  | 1,000,000 ns      |
| 磁盘寻道 (Disk seek)              | 5,000,000 ns      |
| 从磁盘顺序读取 1MB                   | 10,000,000 ns     |
| 发送数据包 CA→Netherlands→CA       | 150,000,000 ns    |

你可能也想追踪与你系统相关的**高层**操作的估算成本，例如 SQL 点查（Point read）的粗略成本、与云服务交互的延迟、渲染一个简单 HTML 页面所需时间。如果不知道各操作的相对成本，就无法做像样的封底计算。

### 示例：对十亿个 4 字节数字进行快速排序的时间

粗略近似下，一个优秀的快速排序需要对大小为 $N$ 的数组做约 $\log(N)$ 次遍历。每次遍历中，数组内容从内存流式传入处理器缓存，分区代码把每个元素与枢轴（Pivot）比较一次。把主要成本加起来：

1. **内存带宽：** 数组占 4GB（4 字节 × 10 亿）。假设每核心内存带宽约 16GB/s，则每次遍历约 0.25 秒。$N \approx 2^{30}$，约需 30 次遍历，故内存传输总成本约 **7.5 秒**。
2. **分支预测错误：** 总共约 $N \times \log(N) \approx 300$ 亿次比较。假设其中一半（150 亿次）预测错误，乘以每次 5 ns，得 **75 秒**。此分析假设预测正确的分支是免费的。

相加得估算值约 **82.5 秒**。

若要进一步考虑处理器缓存：假设有 32MB 的 L3，且 L3→处理器的传输成本可忽略。L3 可容纳约 $2^{23}$ 个数字（32MB ÷ 4 字节 = 8M），因此最后 **22** 次遍历可在驻留 L3 的数据上进行（倒数第 23 次遍历把数据带入 L3，其余在其上操作）。这把内存传输成本从 7.5 秒（30 次 4GB 传输）降到 **2.5 秒**（10 次 4GB 传输，16GB/s）。此时分支预测错误仍是主要成本，所以这步细化其实非必需，此处只作示例。

> Rust 备注：上面用什么语言无关。真要在 Rust 里排序，`slice::sort_unstable`（introsort/pdqsort）通常就是这个量级的代表；分支预测错误正是 pdqsort 等“无分支分区”优化要攻击的成本项。

### 示例：生成含 30 个图像缩略图的网页时间

假设原图存于磁盘，每张约 1MB。比较两种设计：

1. **串行设计：** 逐个读取 30 张图并生成缩略图。每次读 = 一次寻道 + 一次传输 = 5ms + 10ms = 15ms/图，30 张共 **450ms**。
2. **并行设计：** 假设图片均匀分布在 $K$ 个磁盘上并行读取。资源使用估算不变，但延迟大致下降 $K$ 倍（忽略方差，比如偶尔某个磁盘上的图片超过一张）。因此在拥有数百个磁盘的分布式文件系统上，预期延迟降至约 **15ms**。

再看一种变体：所有图片都在单块 SSD 上。此时每张图约 20µs 寻道 + 1ms 传输，总共约 **30ms**。

> Rust 备注：并行版在 Rust 里的惯用实现是 `rayon` 的 `par_iter()`，或 `tokio` 的并发 I/O（`join_all` / `FuturesUnordered`）。

## 测量 (Measurement)

上一节讲的是编写代码时如何思考性能而不必测量。但当你真正开始改进、或遇到性能与简洁性的权衡时，你会想测量或估算潜在收益。**能有效测量，是做性能工作最想拥有的首要能力。**

顺带一提，对不熟悉的代码做剖析，也是了解代码库结构与运行方式的绝佳途径：检查动态调用图中重度参与的例程源码，能让你对“运行时发生了什么”有高层认识，从而更有信心去改动陌生代码。

### 剖析工具和技巧

原文首推 **pprof**（高层信息好、本地和生产都易用），更细节的洞察可用 **perf**。在 Rust 生态里，对应的常用组合是：

* `cargo flamegraph`（基于 perf/dtrace，直接出火焰图）；
* `samply`（跨平台采样剖析，产出可在 Firefox Profiler 里看的 profile）；
* `pprof-rs`（在进程内采样，产出 pprof 格式，适合生产内嵌）；
* 底层仍可直接用 `perf` / `Instruments`（macOS）。

一些剖析技巧：

* 用带有合适调试信息与优化标志的**生产构建**（Rust：`--release` 且在 `Cargo.toml` 里开 `[profile.release] debug = true`，保留符号）。
* 尽量为要改进的代码写**微基准测试（Microbenchmark）**。Rust 用 `criterion`（稳定版）或 `#[bench]`（nightly）。微基准能缩短周转、验证收益、防回退，但可能不代表完整系统性能。
* 用基准库输出**性能计数器**读数，兼顾精度与对程序行为的洞察（`criterion` 可配合 `perf` 计数器）。
* **锁竞争**通常人为压低 CPU 使用率；一些互斥锁实现支持竞争剖析（Rust：`parking_lot` + 相关工具，或 `tracing` 埋点）。
* ML 性能工作用 ML profilers。

### 当剖析结果扁平（Flat）时该怎么办

当所有“低垂果实”都摘完后，CPU 剖析常常是扁平的（没有明显罪魁）。此时考虑：

* **不要低估许多微小优化的价值！** 在一个子系统里做二十次独立的 1% 改进往往完全可行，累计相当可观（这类工作依赖稳定、高质量的微基准）。
* **查找调用栈顶端附近的循环**（火焰图有帮助）。循环或其调用的代码也许能重构得更高效。例如某代码最初通过逐条遍历输入的节点和边来增量构图，后改为一次性传入整个输入构图，消除了每条边上的一堆内部检查。
* **退一步找调用栈更上层的结构性变更**，而非只盯微优化——“算法改进”一节的技术在此有用。
* **寻找过度通用的代码**，用定制或更底层的实现替换。例如若只需简单前缀匹配，就别用正则。
* **减少分配次数**：取分配剖析（allocation profile），干掉贡献最大的分配。效果有二：(1) 直接减少分配器（GC 语言里还有 GC）的时间；(2) 通常减少缓存未命中——长期运行的程序里每次分配往往落到不同缓存行。
* **收集其他类型剖析**，尤其是基于硬件性能计数器的，可指出高缓存未命中率的函数。

## API 设计考量 (API Considerations)

下面一些技术需要改动数据结构和函数签名，可能破坏调用者。尽量把性能改进封在封装边界内、不动公共接口。如果你的**模块较深（Deep Modules）**（通过窄接口访问显著功能），这会更容易。

广泛使用的 API 面临加功能的巨大压力。加新功能要小心：它们会限制未来实现，并为不需要该功能的用户增加不必要成本。例如很多容器承诺**迭代器/指针稳定性**，这在典型实现中显著增加分配次数，即便许多用户并不需要。

> Rust 备注：这条在 Rust 里格外具体。`Vec<T>` 扩容会移动元素、使已有 `&T` 失效（无稳定性），换来连续存储与最少分配；如果调用者确实需要地址稳定，才用 `Vec<Box<T>>`、`typed-arena` 或 `id-arena`（用索引而非引用）。借用检查器会在编译期逼你直面这个取舍，而不是运行期踩坑。

### 批量 API (Bulk APIs)

提供批量操作以减少昂贵的 API 边界跨越，或利用算法改进。

* **添加批量 `lookup_many` 接口。** 除减少跨越外，还能简化签名：若客户端只需知道“是否所有键都找到”，返回 `bool` 即可，不必给每个键一个 `Result`。

```rust
// 单次版：每次都过一遍查找逻辑
fn lookup(&self, key: Key) -> Option<Val>;

// 批量版：一次处理一批，返回是否全部命中
fn lookup_many(&self, keys: &[Key], out: &mut Vec<Val>) -> bool;
```

* **添加批量 `delete_refs` 以摊销锁开销。** 在锁内逐个删 vs 取一次锁删掉全部：

```rust
// 反例：每个 ref 都进出一次锁
for r in refs { store.lock().unwrap().delete(r); }

// 正例：取一次锁批量处理
let mut guard = store.lock().unwrap();
for r in refs { guard.delete(*r); }
```

* **用 Floyd 建堆法高效初始化。** 批量建堆是 $O(N)$，而逐个 `push` 并每次上浮维护堆序是 $O(N \log N)$。Rust 里 `BinaryHeap::from(vec)` 正是走的 $O(N)$ 建堆（`heapify`），而循环 `push` 是 $O(N \log N)$——优先用 `from`。

有时难以改造调用者直接用批量 API。此时可在内部用批量 API 并缓存结果供后续单次调用：

* **缓存块解码结果。** 每次查找都要解码整块的 K 个条目——把解码后的条目缓存起来，后续查找命中缓存。

### 视图类型 (View types)

对函数参数，优先用视图类型，除非要转移数据所有权。视图减少复制，并让调用者自选容器类型。

| C++ | Rust |
| --- | --- |
| `std::string_view` | `&str` |
| `absl::Span<T>` | `&[T]` |
| `absl::FunctionRef<R(Args…)>` | `&dyn Fn(…) -> R` 或 `impl Fn` |

```rust
// 反例：强制调用者交出 Vec 的所有权，且逼死容器类型
fn sum(xs: Vec<i64>) -> i64 { xs.iter().sum() }

// 正例：接受切片视图；调用者用 Vec、SmallVec、数组、甚至 &[T; N] 都行
fn sum(xs: &[i64]) -> i64 { xs.iter().sum() }
```

一个调用者可能传 `Vec<T>`，另一个传 `SmallVec<[T; N]>`——`&[T]` 视图对两者一视同仁，无需复制。

### 预分配/预计算参数

对频繁调用的例程，允许上层传入它们已有的、被调例程需要的数据结构或信息，可避免底层被迫分配临时结构或重算已有信息。

* **加一个允许客户端传入已有 `WallTime` 的变体**，避免底层反复调用 `now()`：

```rust
fn record_rpc(&self, ...);                 // 内部自己取时间
fn record_rpc_at(&self, ..., now: Instant); // 复用调用者已有的时间戳
```

也可以让调用者传入一个可复用的输出缓冲（`&mut Vec<T>` / `&mut String`），底层往里写而不新分配——见后文“重用临时对象”。

### 线程兼容 (Thread-compatible) vs 线程安全 (Thread-safe) 类型

类型可以是线程兼容的（外部同步）或线程安全的（内部同步）。**大多数通用类型应做成线程兼容的**，这样不需要同步的调用者就不必为此付费。

* **让类型线程兼容，因为调用者已在外部同步。** 若每个方法都自己加锁、而调用者外层也持锁，就是双重开销。

> Rust 备注：这条几乎是 Rust 的“母语”。一个普通 `struct` 默认就是线程兼容的——修改需要 `&mut self`，借用检查器在编译期保证独占，零运行期成本。需要跨线程共享可变状态时，调用者显式用 `Mutex<T>` / `RwLock<T>` 从**外部**包起来；`Send`/`Sync` 标记则由编译器推导谁能安全跨线程。也就是说，你几乎从不需要在类型内部塞锁。

然而，若类型的典型用法本就需要同步，则倾向把同步移进类型内部。这样可在必要时调整同步机制以提升性能（例如分片降竞争），而不影响调用者——Rust 里对应 `dashmap`（内部分片的并发 map）这类“内部同步”容器。

## 算法改进 (Algorithmic Improvements)

最关键的性能机会来自算法改进，例如把 $O(N^2)$ 变 $O(N \log N)$ 或 $O(N)$、避免潜在指数行为等。这类机会在稳定代码里少见，但写新代码时值得关注。

* **以逆后序（Reverse post-order）向循环检测结构添加节点**，避免每条边都做昂贵检查。
* **用更好的算法替换互斥锁内置的死锁检测。** 新算法基于 Pearce 等人的论文，空间从 $O(|V|^2)$ 降到 $O(|V|+|E|)$，快约 50 倍，可扩展到数百万个锁。
* **用哈希表（$O(1)$ 查找）替换 `IntervalMap`（$O(\log N)$ 查找）。** 如果只是合并相邻块，`HashMap` 可能就够。
* **用哈希查找（$O(N)$）替换排序列表求交（$O(N \log N)$）。** 检测两节点是否共享公共源。
* **实现良好的哈希函数，使操作 $O(1)$ 而非 $O(N)$。** 避免哈希碰撞退化成链表查找。Rust 里：非加密场景可换 `rustc-hash::FxHashMap` 或 `ahash`（比默认的 SipHash 快很多），但要注意别把它们用在需要抗 HashDoS 的对外输入上。

## 更好的内存表示 (Better memory representation)

仔细考虑重要数据结构的内存与缓存占用，常能带来巨大节省。以下结构都着眼于用**更少的缓存行**支撑常见操作，好处是 (a) 避免昂贵缓存未命中，(b) 减少内存总线流量——这不仅加速本程序，还惠及同机其他程序。

### 紧凑数据结构 (Compact data structures)

对访问频繁、或占内存大头的数据用紧凑表示。

* **内存布局：** 重排字段以减少对齐填充；用更小的数值类型。
* **字段排序：** 常一起访问的字段放一起；把热的只读字段与热的可变字段分开（避免伪共享导致的缓存失效）；冷数据挪到结构体末尾或通过间接访问。
* **位/字节级编码：** 仅在数据被封装于经良好测试的模块内、且内存节省显著时才这么做。

```rust
// 反例：字段顺序导致大量对齐填充，且用了过宽的类型
struct Node {
    flag: bool,   // 1 字节 + 7 填充
    id: u64,      // 8 字节
    kind: u8,     // 1 字节 + 7 填充
}                 // sizeof 约 24

// 正例：大字段在前、同类相邻、用最小够用的类型
struct Node {
    id: u32,      // 索引足够用 32 位就别用 64（见下节）
    kind: u8,
    flag: bool,
}                 // 紧凑得多
```

Rust 备注：默认布局编译器会重排字段来省填充（不像 C 默认按声明序）；需要精确控制时用 `#[repr(C)]` 固定顺序、`#[repr(packed)]` 去填充（有未对齐访问风险，慎用）。位域可用 `bitflags` crate 或手写位运算，或 `bitfield` crate。

### 索引代替指针 (Indices instead of pointers)

64 位机上指针占 64 位。指针密集的结构里，可改用对数组 `[T]` 的整数索引。若索引数量足够小（32 位以内），引用不仅更小，而且 `[T]` 的所有元素连续存储，通常带来更好的**缓存局部性**。

```rust
// 反例：指针追逐（pointer chasing），每个节点一次堆分配，散落各处
struct Node { next: Option<Box<Node>>, val: i32 }

// 正例：所有节点放进一个 Vec，用 u32 索引串联；连续、紧凑、缓存友好
struct Node { next: Option<u32>, val: i32 }
struct Graph { nodes: Vec<Node> } // 用 nodes[i as usize] 访问
```

> Rust 备注：这在 Rust 里是双赢——它既是缓存优化，又是**绕开借用检查器**构建图/树（含环、共享、双向引用）的标准手法，比 `Rc<RefCell<Node>>` 更省、更简单。生态里 `id-arena`、`slotmap`、`generational-arena` 就是把“Vec + 索引”封装好并处理了失效问题（用带 generation 的 key 防悬垂索引）。

### 批处理存储 (Batched storage)

避免为每个元素单独分配对象的结构（如 C++ 的 `std::map`）。改用把多个元素在内存中紧邻存放的分块/扁平表示。

| 逐元素分配（避免） | 批处理存储（优先） |
| --- | --- |
| `std::map` / `std::unordered_map` | `std::collections::HashMap`（底层 hashbrown，扁平） |
| 链式结构 | `Vec<T>`、`BTreeMap`（分块节点） |

Rust 备注：std 的 `HashMap` 本就是扁平（Swiss Tables）实现，天然满足这条；`BTreeMap` 的节点是多元素分块的，也比“每键一分配”的红黑树缓存友好。真正要警惕的是自己用 `Vec<Box<T>>` / 链表把每个元素单独装箱。

### 内联存储 (Inlined storage)

一些容器针对“存少量元素”做了优化，元素少时避免堆分配。

* C++ `absl::InlinedVector<T, N>` → Rust `smallvec::SmallVec<[T; N]>`（超过 N 个再溢出到堆）或 `arrayvec::ArrayVec<T, N>`（定长，满了报错，永不分配）。

```rust
use smallvec::{SmallVec, smallvec};
// 90% 情况 ≤ 4 个元素时，完全不碰堆
let mut v: SmallVec<[u32; 4]> = smallvec![1, 2, 3];
```

* **警告：** 若 `size_of::<T>()` 很大，内联后备存储会很大，可能撑爆栈或让对象过大。

### 不必要的嵌套映射 (Unnecessarily nested maps)

有时可用“复合键的单层映射”替换嵌套映射，显著降低查找/插入成本。

```rust
// 反例：嵌套 map，两次哈希 + 内层 map 的额外分配
let outer: HashMap<A, HashMap<B, C>> = ...;

// 正例：复合键单层 map，一次哈希、一次分配
let flat: HashMap<(A, B), C> = ...;
```

（C++ 里对应 `btree<a, btree<b,c>>` → `btree<pair<a,b>, c>`。）

* **警告：** 若第一层键很大，或需要利用第一层键的共享结构，嵌套映射可能更好。

### 区域内存 (Arenas)

Arena 能降低分配成本，还能把独立分配的项紧挨着打包（常落在更少缓存行里），并消除大多数析构成本。对“含许多子对象的复杂数据结构”最有效。

```rust
use bumpalo::Bump;
let arena = Bump::new();
let a = arena.alloc(Node { .. }); // 众多小对象共享一次大块分配
let b = arena.alloc(Node { .. }); // 紧邻 a；arena drop 时一次性释放
```

> Rust 备注：`bumpalo`（bump 分配）、`typed-arena`（同类型对象、返回 `&mut T`）、`id-arena`/`slotmap`（返回索引/key）各有侧重。Arena 里的对象默认**不逐个析构**（正是“消除析构成本”的来源），所以别往里放持有 `Drop` 资源（文件句柄等）的类型。

### 数组代替映射 / 位向量代替集合

* 若映射的**域很小或是枚举**，用数组代替 `HashMap`：`enum Color { Red, Green, Blue }` 的计数用 `[u32; 3]`，`color as usize` 索引，零哈希零分配。
* 若集合的域可由小整数表示，用**位向量**代替集合。集合运算（并、交）变成高效位运算。

```rust
// 集合 → 位向量：并集/交集变成 | 和 &
use fixedbitset::FixedBitSet;
let mut a = FixedBitSet::with_capacity(1024);
let mut b = FixedBitSet::with_capacity(1024);
a.insert(3); a.insert(700);
b.insert(700);
a.intersect_with(&b); // 位运算，远快于 HashSet 求交
```

（C++ 的 `InlinedBitVector` → Rust 的 `bitvec` / `fixedbitset` / `bit-set`。）

## 减少分配 (Reduce allocations)

内存分配的成本有三：

1. 分配器本身耗时。
2. 新对象可能需要昂贵的初始化，用完还要昂贵的销毁。
3. 每次分配往往落在新缓存行上——分散在许多独立分配上的数据，比集中在少数分配上的有更大缓存占用。

### 避免不必要的分配

* **Lazy initialization：** 只在需要时才分配。Rust：`std::sync::LazyLock` / `OnceLock`（1.80+），或 `once_cell`。

```rust
use std::sync::LazyLock;
static TABLE: LazyLock<Vec<u32>> = LazyLock::new(|| compute_table());
```

* **用静态分配的零向量，而不是分配并清零。** Rust：`const ZEROS: [u8; N] = [0; N];` 或 `static`，直接借用它。
* **优先栈分配而非堆分配**（当对象生命周期受限于作用域时）。Rust 里默认就是栈分配，除非你显式 `Box`/`Vec`/`String`；这条提醒你别无谓地装箱。

### 调整容器大小或预留 (Resize or reserve containers)

已知向量（或其他容器）最大或预期最大大小时，预先调整后备存储大小。

```rust
// 反例：从 0 开始逐个 push，可能触发多次 realloc + 拷贝
let mut v = Vec::new();
for i in 0..n { v.push(i); }

// 正例：一次预留到位
let mut v = Vec::with_capacity(n);
for i in 0..n { v.push(i); }
// 或直接 let v: Vec<_> = (0..n).collect();  // collect 会用 size_hint 预留
```

* **警告：** 不要每次只增长一个元素地反复 `reserve`/`resize`，那会导致二次方行为。（`Vec::push` 自身的**几何**扩容是安全的；手动逐元素 `reserve(len+1)` 才是坑。）

### 尽可能避免复制

* **优先移动（Move）而非复制（Copy）。**

> Rust 备注：这在 Rust 里是**默认行为**——赋值/传参默认移动所有权，深拷贝必须显式 `.clone()`。所以这条建议在 Rust 里翻译成：“别乱写 `.clone()`”。看到热路径上的 `.clone()`，先问是不是可以改成借用 `&T` 或移动。

* 如果生命周期允许，在临时结构里存**索引或引用**而非对象副本（见“索引代替指针”）。
* **避免通过 gRPC 接收张量时的额外复制**——Rust 里用 `bytes::Bytes`（引用计数、可切片、零拷贝）承载网络负载。
* **不需要稳定性时用 `sort_unstable` 而非 `sort`**：`slice::sort` 稳定但需要 $O(N)$ 辅助内存（内部复制），`sort_unstable` 原地、更快。

```rust
xs.sort_unstable();              // 无需稳定 → 更快、免辅助分配
xs.sort();                       // 需要稳定 → 有内部缓冲
```

### 重用临时对象

把循环内声明的容器/对象提升到循环外，跨迭代重用，避免反复构造/析构/扩容。Rust 的关键是 `.clear()` **保留已分配容量**：

```rust
// 反例：每次迭代都新建 + 释放一个 String
for line in lines {
    let mut buf = String::new();
    render_into(&mut buf, line);
    emit(&buf);
}

// 正例：循环外建一次，clear() 复用底层内存（容量保留）
let mut buf = String::new();
for line in lines {
    buf.clear();          // 长度归零但不释放容量
    render_into(&mut buf, line);
    emit(&buf);
}
```

（C++ 里对应“把 protobuf 消息变量提到循环外”“反复序列化到同一个 `std::string`”——因为它们会保留已分配内存。Rust 的 `String`/`Vec` 同理。）

## 避免不必要的工作 (Avoid unnecessary work)

### 常见情况的快速路径 (Fast paths for common cases)

代码常写成覆盖所有情况，但某些子集更简单也更常见。

* **让快速路径覆盖更多常见情况。** 例如末尾单个 ASCII 字节也走快路径，而不仅处理 4 字节倍数。
* **`SmallVec` 的更简单快速路径**（元素少时不碰堆）。
* **Varint 解析器的快速路径：** 专门处理 1 字节情况。
* **RPC 统计若无错误发生，跳过大量工作。**
* **指纹整串之前，先对首字节做一次数组查找**（过滤器模式，先廉价排除大多数不匹配）。

```rust
// 先用首字节数组查找做廉价过滤，命中才做昂贵的整串指纹
if !MAYBE_TABLE[s.as_bytes()[0] as usize] {
    return false;               // 冷路径都不用进
}
expensive_fingerprint(s)
```

### 预先计算昂贵的信息 (Precompute expensive information once)

* **预计算执行节点属性**（如 TF 图节点）。
* **预计算 256 元素数组，在初始化时构好、后续直接查。**
* **通用建议：** 在**模块边界**检查格式错误的输入，而不是在内部反复检查。

```rust
// 编译期就把 256 项查找表算好
const PARITY: [u8; 256] = build_parity_table();
```

### 将昂贵的计算移出循环

* **把边界/不变量计算移出循环**（loop-invariant code motion）：

```rust
// 反例：每次迭代都重算 xs.len() 相关的界
for i in 0..xs.len() { let n = expensive_bound(&cfg); use(i, n); }

// 正例：循环外算一次
let n = expensive_bound(&cfg);
for i in 0..xs.len() { use(i, n); }
```

### 延迟昂贵的计算 (Defer expensive computation)

* **直到真正需要时才计算**（如按需调用 `get_sub_sharding`）。
* **别急切更新统计信息；按需计算。** Rust：用 `OnceCell`/惰性字段缓存首次计算结果。

### 特化代码 (Specialize code)

性能敏感的特定调用点可能用不上通用库的全部通用性。

* **自定义打印代码比通用格式化快数倍**（原文：Histogram 的自定义打印比 `sprintf` 快 4 倍）。Rust：热路径上避免 `format!`，直接 `write!` 进预留好的 `String`，或手写数字转字符串。
* **为不同日志级别加特化**以提速并减小代码体积。
* **尽可能用简单前缀匹配替换正则**：Rust 里若 `s.starts_with("foo")` 就够，别上 `regex`。
* **用轻量拼接替代重量格式化**：格式化 IP 地址等，用 `push_str`/`write!` 而非 `format!` 的完整解析路径。

### 使用缓存避免重复工作

* **基于大对象的预计算指纹做缓存**（例如对大 proto 的指纹缓存，避免重复序列化/比较）。Rust：`HashMap<Key, Cached>` 或 `memoize`/`cached` crate。

### 让编译器的工作更轻松 (Make the compiler's job easier)

* **避免在热点函数里做函数调用**，让编译器省掉栈帧设置——Rust 用 `#[inline]` 提示，极热的小函数可 `#[inline(always)]`。
* **把慢路径代码挪到单独的尾调用函数里**，并标 `#[cold]` + `#[inline(never)]`，让热路径的指令缓存更干净：

```rust
#[inline]
fn get(&self, k: Key) -> Val {
    match self.fast_lookup(k) {
        Some(v) => v,
        None => self.slow_path(k),   // 冷路径拆出去
    }
}

#[cold]
#[inline(never)]
fn slow_path(&self, k: Key) -> Val { /* 少见、复杂 */ }
```

* **用前先把少量数据拷进局部变量。** 这让编译器确信没有与其他数据的**别名（aliasing）**，从而改善自动向量化和寄存器分配。Rust 的 `&mut` 独占借用本就给编译器强别名保证（对应 LLVM `noalias`），这也是 Rust 常能自动向量化得不错的原因之一。
* **手动展开非常热的循环**（如 CRC 计算）——Rust 里优先用 `chunks_exact(N)` 让编译器一次处理 N 个元素，多数情况下比手写展开更干净且能触发 SIMD：

```rust
let mut acc = 0u64;
let mut it = bytes.chunks_exact(8);
for c in &mut it { acc ^= u64::from_le_bytes(c.try_into().unwrap()); } // 一次 8 字节
for &b in it.remainder() { acc ^= b as u64; }                          // 尾巴
```

### 减少统计收集成本

平衡统计的效用与维护成本。

* **完全丢弃无用统计。**
* **采样（Sampling）：** 只维护样本的统计。
* **降低采样率并加快采样决策。**

### 避免在热点代码路径上记录日志

即便未启用的日志语句也可能有成本（加载并比较级别）。

* **从内存分配器核心里移除日志。**
* **在嵌套循环外预先算好“日志是否启用”。** Rust：`log`/`tracing` 的 `log_enabled!(Level::Debug)` 提前判一次；`tracing` 的编译期 `max_level_*` feature 可在 release 里彻底编掉低级别日志。

## 代码体积考量 (Code size considerations)

代码体积大意味着更长的编译/链接、臃肿的二进制、更多内存、更大的指令缓存（icache）压力。

* **削减通常被内联的代码：** 广泛调用的函数结合内联会显著撑大代码。把慢路径挪进非内联函数（`#[inline(never)]` / `#[cold]`）。
* **谨慎内联：** 内联有时增大体积却无性能回报——不要滥用 `#[inline(always)]`。
* **减少泛型单态化膨胀**（这是 Rust 版最重要的一条）：
  * 把泛型参数换成运行期参数（若运行期检查成本低）；
  * 把大段与类型无关的代码从泛型函数移到非泛型的“内核”函数，让泛型外壳只做很薄的转发（避免每个 `T` 都复制一大坨机器码）。

```rust
// 反例：整段逻辑随每个 T 单态化，二进制膨胀
fn write_all<W: Write>(w: &mut W, data: &[u8]) { /* 大量代码 */ }

// 正例：泛型薄壳 + 非泛型内核，机器码只生成一份
fn write_all<W: Write>(w: &mut W, data: &[u8]) { write_all_inner(w as &mut dyn Write, data); }
fn write_all_inner(w: &mut dyn Write, data: &[u8]) { /* 大量代码，仅一份 */ }
```

* **减少容器操作：** 把一串 `insert` 合成一次批量插入（`HashMap::extend(iter)`）。

## 并行化和同步 (Parallelization and synchronization)

### 利用并行性

现代机器核多，昂贵工作可并行加速。

* 原文例：四路并行让令牌编码提速约 3.6 倍；并行让解码提速 5 倍。
* Rust 惯用法：`rayon` 的 `par_iter()` 几乎零改造地把数据并行铺开：

```rust
use rayon::prelude::*;
let sums: Vec<_> = chunks.par_iter().map(|c| encode(c)).collect();
```

* **警告：** 务必测量——若没有空闲 CPU、或内存带宽已饱和，并行可能无益甚至有害。

### 摊销锁获取 (Amortize lock acquisition)

* **取一次锁释放整棵查询节点树，而不是为每个节点重新取锁**（见前文 `delete_refs` 例）。

### 保持临界区简短

避免在临界区内做昂贵的事（RPC、文件访问）。

* **减少临界区内接触的缓存行数。**
* **别在持锁时做 RPC。** Rust：把要发出的数据在锁内快速拷出，`drop(guard)` 后再做 I/O；异步代码里尤其别跨 `.await` 持有 `std::sync::Mutex`（该用 `tokio::sync::Mutex` 或干脆缩短持锁范围）。

### 通过分片减少竞争 (Reduce contention by sharding)

* **把缓存分成 16 片**，多线程下吞吐约翻倍。
* Rust：直接用 `dashmap`（内部分片的并发 map），或自己 `Vec<Mutex<Shard>>` 按 key 哈希选片。
* **注意：** 确保分片选择用的信息与哈希表内部哈希值不冲突，以免倾斜。

### 减少伪共享 (Reduce false sharing)

不同线程访问不同的可变数据时，把它们放到不同缓存行。

```rust
use crossbeam_utils::CachePadded;
// 每个计数器独占一条缓存行，避免线程间互相踢缓存
let counters: Vec<CachePadded<AtomicU64>> = ...;
```

（C++ 的 `alignas(64)` → Rust 的 `#[repr(align(64))]` 或 `crossbeam_utils::CachePadded<T>`。）

### 考虑无锁方法 (Consider lock-free approaches)

* 用无锁 map 管理 RPC 通道缓存；用“固定词典 + 无锁哈希”加速合法性校验。
* Rust：`arc-swap`（整体配置的读多写少无锁替换）、`crossbeam`（无锁队列/栈/epoch 回收）、`dashmap`。无锁很难写对，优先复用成熟 crate。

## Protocol Buffer 建议

Protobuf 方便但有性能成本。把数据从 protobuf 转成原生 `Vec<T>` 可能有 20 倍加速。（Rust 侧用 `prost` 或 `protobuf` crate。）

* **别不必要地用 protobuf。** 若数据从不序列化/解析，就别用它，直接用普通 struct。
* **避免不必要的消息层次。** 扁平结构通常更好。
* **给高频字段用小字段号。** 1–15 占 1 字节。
* **仔细选整数类型：** `int32/64` vs `fixed32/64` vs `sint32/64`（值分布决定谁更省）。
* **proto2 用 `[packed=true]`；proto3 默认已打包。**
* **二进制数据与网络大值用 `bytes` 而非 `string`**，避免 UTF-8 校验。Rust `prost` 里可让 `bytes` 字段映射为 `bytes::Bytes` 实现零拷贝。
* **大字段考虑零拷贝承载**：C++ 的 `string_type = VIEW` / `Cord` → Rust 的 `bytes::Bytes`。
* **C++ 里用 protobuf arena 摊销分配**——Rust 侧无直接对应；`prost` 生成普通 struct，减分配主要靠复用消息对象（`clear()` 后重填）和 `Bytes` 零拷贝。
* **保持 .proto 文件小**，避免巨大依赖。

## Rust 特定建议（替换原文“C++ 特定建议”）

原文这节推荐了一批 Abseil/Google 容器。下面给出 Rust 生态里“开箱即用或一行依赖”的对应选择：

* **`std::collections::HashMap`（Swiss Tables）：** std 自 1.36 起底层即 hashbrown，SIMD 并行比较哈希字节，通常优于其他哈希表——你**默认用的就是**原文推崇的 `absl::flat_hash_map`。
* **更快的哈希器：** 非加密、可信输入场景，把 hasher 换成 `rustc-hash::FxHashMap` 或 `ahash::AHashMap`，比默认 SipHash 快不少（代价是不抗 HashDoS）。
* **`BTreeMap`/`BTreeSet`：** 缓存友好的有序容器，对应 `absl::btree_map/set`。
* **`bitvec` / `fixedbitset`：** 对应 `InlinedBitVector`，优于 `Vec<bool>`。
* **`SmallVec` / `ArrayVec`：** 小向量优化，对应 `absl::InlinedVector`。
* **`smallmap` / 线性小 map：** 元素极少时，线性扫描的 `Vec<(K,V)>` 可能比哈希表更快（对应 `gtl::small_map`）。
* **限制 `Result`/错误对象在热路径上的构造：** 对应原文“限制 `absl::Status`/`StatusOr`”。热路径上：
  * 用 `Option<T>` 代替 `Result<T, E>`（当“没有”不是错误时）；
  * 让错误类型小且廉价（避免每次都 `format!` 一个 `String` 错误——用 `&'static str`、错误码枚举，或 `thiserror` 的零成本变体）；
  * 给遍历/查找提供不返回错误的“NoStatus”版本。

```rust
// 反例：热路径每次都构造带堆分配 String 的错误
fn lookup(&self, k: Key) -> Result<Val, String> { ... }

// 正例：命中/未命中不是“错误”，用 Option，零错误对象构造
fn lookup(&self, k: Key) -> Option<Val> { ... }
```

## 批量操作 (Bulk operations)

* **`HashMap` 用单条 SIMD 指令从一组槽位里比较每个键的一个哈希字节**——std/hashbrown 已内建，你无需手写。
* **一次处理许多字节再统一修正，而不是每字节都判断该做什么**（循环展开 + 批处理）：Rust 用 `chunks_exact(N)`、`bytemuck`（安全地把 `&[u8]` 重解释为 `&[u64]`）、或 `std::simd`（nightly）。
* **一次解码一组整数（如 4 个）比逐个解码快**——批量 varint 解码同理，把分支预测和加载摊销到一批上。

## 扩展阅读

* **Optimizing software in C++** — Agner Fog（虽是 C++，但缓存/流水线/分支部分与语言无关，Rust 同样受用）。
* **Understanding Software Dynamics** — Richard L. Sites。
* **Computer Architecture: A Quantitative Approach** — Hennessy and Patterson。
* Rust 侧补充：**The Rust Performance Book**（nnethercote，`nnethercote.github.io/perf-book`）、`rayon` / `criterion` / `hashbrown` / `bumpalo` / `smallvec` 各自的 docs.rs 文档。

<!-- obsidian-publish-source: 40_Programming_SE_编程与软件工程/30_Debug_Performance_调试与性能/Performance Optimization 性能优化/Performance Hints/「译」Performance Hints (Rust 版).md -->