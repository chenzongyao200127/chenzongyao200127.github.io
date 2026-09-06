> 原文链接：https://abseil.io/fast/hints.html#performance-hints
> 姊妹篇：[[「译」Performance Hints]]（C++ 原味版）。
>
> 本篇是**基于逐字原文（abseil.github.io 源码 fast/hints.md，2025/12/16 版）的完整忠实翻译**：保留原文全部章节、子节与每一条 CL 示例（含其关键基准数字），并把其中的 C++ 类型与代码改写成 Rust 等价物。所有 Rust 映射均为译者添加、以引注或 "Rust 备注 " 标出；技术判断与数字一律以原文为准。

## 关于这一版

原文的通用原则（估算、测量、算法改进、缓存友好、批量摊销、快速路径）与语言无关，真正 "C++ 味 " 的是那些具体类型（`std::string_view`、`absl::InlinedVector`、`absl::flat_hash_map`、`StatusOr` 等）与 Google 内部库抽象。本篇保留原文的章节结构、每条 CL 示例及其数字，只把这些具体载体换成 Rust 世界里的对应物，并在有意思处点出 Rust 与 C++ 的差异——很多在 C++ 里需要 " 刻意为之 " 的优化，在 Rust 里恰是默认行为（移动语义、`&mut` 独占借用），这本身就是理解 " 机械同理心 " 的好切口。

Rust 代码片段追求 " 读得懂 + 说清动因 "，不保证可直接 `cargo build`；涉及的第三方 crate 会标出名字，方便去 docs.rs 查。

### C++ → Rust 概念映射总表

| 原文 C++ 载体 | Rust 等价物 | 说明 / 动因 |
| --- | --- | --- |
| `std::string_view` | `&str` | 借用而不拥有，零拷贝视图 |
| `absl::Span<T>` / `std::span<T>` | `&[T]` / `&mut [T]`（切片） | 语言内建，调用者自选底层容器 |
| `absl::FunctionRef<R(Args…)>` | `&dyn Fn(Args) -> R` 或 `impl Fn(…)` | 不拥有闭包，避免装箱分配 |
| `std::vector<T>` | `Vec<T>` | 堆上连续存储 |
| `absl::InlinedVector<T, N>` | `smallvec::SmallVec<[T; N]>` / `arrayvec::ArrayVec<T, N>` | 小容量内联栈上，省堆分配 |
| `absl::FixedArray<T>` | `Box<[T]>` / `Vec<T>`（`with_capacity` 后不增长） | 定长堆数组 |
| `gtl::vector32<T>` | Rust 无直接对应；用 `u32` 索引下标即可 | 用 32 位而非 64 位的 size/capacity 省内存 |
| `absl::flat_hash_map/set`（Swiss Tables） | `std::collections::HashMap/HashSet` | std 自 1.36 起底层即 hashbrown（Swiss Tables 的 Rust 移植），SIMD 探测已内建 |
| `absl::btree_map/set` | `std::collections::BTreeMap/BTreeSet` | 缓存友好的有序容器 |
| `std::map` / `std::unordered_map` | `BTreeMap` / `HashMap` | — |
| `gtl::small_map` | 元素极少时用 `Vec<(K,V)>` 线性扫描 | 小数据集免哈希开销 |
| `gtl::small_ordered_set` | 元素极少时用有序 `Vec<T>` + 二分 | 小有序集合 |
| `gtl::intrusive_list` | 侵入式链表：`intrusive-collections` crate，或 `Vec`+ 索引 | 省每元素一次分配与一条缓存行的间接 |
| `std::vector<bool>` / `util::bitmap::InlinedBitVector` | `bitvec`、`fixedbitset`、`bit-set` crate | 位打包，集合运算变位运算 |
| Arena（`google::protobuf::Arena` 等） | `bumpalo::Bump`（不逐个析构）、`typed-arena`（会析构内容）、`id-arena`/`slotmap` | 批量分配、（视 crate）批量释放；注意各 crate 的 Drop 语义不同 |
| 用索引代替指针 | `Vec<T>` + `u32` 索引 / `id-arena` / `slotmap` | Rust 里构图惯用法，同时绕开借用检查器的自引用难题 |
| `absl::Status` / `StatusOr<T>` | `Result<(), E>` / `Result<T, E>`；非错误的 " 缺失 " 用 `Option<T>` | 热路径避免构造重错误对象；让 `E` 廉价 |
| move vs copy | Rust 默认移动；`Copy` 标记廉价按位复制；`.clone()` 显式深拷贝 | " 优先 move" 在 Rust 里是编译器默认，拷贝必须显式写出 |
| `reserve` / `resize` | `Vec::with_capacity` / `Vec::reserve` | — |
| `std::sort` vs `std::stable_sort` | `sort_unstable`（pdqsort）vs `sort`（稳定，需辅助内存） | 不需要稳定性就用 unstable |
| `StrCat` / `StringPrintf` / `sprintf` | `format!` / `write!` / `String::push_str` | 热路径避免 `format!`，直接拼 buffer |
| `alignas(64)` / `ABSL_CACHELINE_SIZE` 防伪共享 | `#[repr(align(64))]`、`crossbeam_utils::CachePadded<T>` | 把互相独立的可变字段隔到不同缓存行 |
| lock-free map / concurrent hash map | `dashmap`（内部分片）；只读多写少整体替换用 `arc-swap` | 分片/无锁并发 |
| 线程兼容 vs 线程安全 | `Send`/`Sync` + 外部 `Mutex`/`RwLock` vs 内部同步 | 借用检查器让 " 线程兼容 " 成为零成本默认 |
| protobuf | `prost` / `protobuf` crate | Rust 里 arena 关联弱 |
| protobuf `Cord`（`[ctype=CORD]`） | `bytes::Bytes`（引用计数共享） | 减少大字段复制 |
| protobuf `string_type = VIEW` | `&[u8]` / `&str`（借用后备缓冲，生命周期系于原缓冲） | 非拥有借用，与 Cord 不同 |
| pprof | `cargo flamegraph`、`samply`、`pprof-rs`、`perf` | Rust 生态采样剖析工具 |
| Lazy init | `std::sync::LazyLock`/`OnceLock`（1.80+）、`once_cell` | 首次使用才初始化 |
| 手动展开 / 批量字节 | 手写展开仍可；或 `chunks_exact(N)` + `bytemuck`；SIMD 用 `std::simd`（nightly）/`std::arch` | 原文强调 " 手动展开非常热的循环 " |
| SIMD 指令 | `std::arch`（intrinsics）、`std::simd`（nightly）、`wide`/`bytemuck` crate | 一次处理多个元素 |
| 缓冲通道做流水线 | `std::sync::mpsc::sync_channel(n)`、`crossbeam-channel` 有界通道 | 有缓冲才能增并行；无缓冲仅作同步 |
| `#[inline]` 控制 | `#[inline]` / `#[inline(always)]` / `#[inline(never)]` / `#[cold]` | 慢路径标 `#[cold]` 拆出内联 |
| 泛型单态化膨胀 | 把大段与类型无关的逻辑抽到非泛型内核函数，泛型外壳只转发 | 对应原文 " 减少模板实例化 " |

---

# 性能优化建议 (Performance Hints) · Rust 版

> 作者：Jeff Dean, Sanjay Ghemawat
> 原始版本：2023/07/27，最后更新：2025/12/16

多年来，我们（Jeff 和 Sanjay）在各种代码片段的性能调优上投入了大量精力。自 Google 成立之初，提升软件性能就至关重要，因为这让我们能为更多用户提供更多服务。本文档旨在总结我们做此类工作时运用的一些通用原则和特定技巧，并挑选有代表性的源代码变更（Change Lists，简称 CLs）作为具体方法的示例。原文的具体建议大多涉及 C++ 类型与 CL，但其通用原则同样适用于其他语言——本篇即把它们落到 Rust 上。

本文档聚焦于单个二进制文件上下文中的通用性能调优，不涵盖分布式系统或机器学习（ML）硬件的性能调优（这些本身就是庞大的领域）。希望这份文档对大家有所帮助。

文档中的许多示例都包含演示相关技巧的代码片段。请注意，部分片段可能提及 Google 内部代码库的各种抽象；如果我们认为这些示例足够独立、即便不熟悉这些抽象也能理解，仍会将其收录。

## 思考性能的重要性 (The importance of thinking about performance)

Knuth 有一句常被断章取义引用的话：*过早优化是万恶之源 (premature optimization is the root of all evil)*。[完整原文](https://dl.acm.org/doi/pdf/10.1145/356635.356640) 是：*" 在大约 97% 的时间里，我们应当忘掉那些小的效率提升：过早优化是万恶之源。然而，我们不应错过那关键的 3% 中的机会。"* 本文讨论的正是那关键的 3%。Knuth 还有另一段更有说服力的话：

> 从示例 2 到示例 2a 的速度提升只有大约 12%，许多人会认为这微不足道。当今许多软件工程师所共有的传统观念主张忽略微观层面的效率；但我认为这只是对他们所见到的、那些因小失大的程序员滥用优化的做法的一种过度反应——这些程序员根本无法调试或维护他们 " 优化过 " 的程序。在成熟的工程学科中，一个轻易可得的 12% 的提升绝不会被视为无关紧要；我相信同样的观点也应在软件工程中占主导地位。当然，对于一次性的工作我不会费心去做这种优化，但当涉及到编写高质量的程序时，我不想把自己局限在那些不允许我获得此类效率的工具上。

许多人会说 " 让我们用尽可能简单的方式把代码写下来，等到能够做剖析时再处理性能问题 "。然而，这种做法往往是错误的：

1.  如果你在开发一个大型系统时无视所有性能问题，最终你会得到一份平坦的剖析结果（flat profile），其中没有明显的热点，因为性能是在各个地方一点一点流失掉的。届时将很难弄清楚该从何处着手进行性能改进。
2.  如果你正在开发一个将被其他人使用的库，那么最终遭遇性能问题的人，很可能是那些难以轻易做出性能改进的人（他们必须理解由其他人／团队编写的代码细节，还得就性能优化的重要性与对方进行协商）。
3.  当一个系统被大量使用时，对它做出重大改动会更加困难。
4.  同时也很难判断是否存在一些可以轻松解决的性能问题，于是我们最终可能采用一些代价高昂的方案，例如过度复制（over-replication）或严重超额配置（severe overprovisioning）某个服务，以应对负载问题。

因此，我们建议：在编写代码时，如果更快的方案不会显著影响代码的可读性／复杂度，就尽量选择更快的那一个。

## 估算 (Estimation)

如果你能培养出一种直觉，去判断你正在编写的代码中性能可能有多重要，你就能做出更明智的决策（例如，为了性能到底值得引入多少额外的复杂度）。以下是在编写代码时估算性能的一些技巧：

* 这是测试代码吗？如果是，你主要需要关心算法和数据结构的渐进复杂度（asymptotic complexity）。（附注：开发周期的时长也很重要，所以要避免编写运行时间很长的测试。）
* 这是特定于某个应用的代码吗？如果是，试着弄清楚这段代码对性能有多重要。这通常并不太难：仅仅弄清楚这段代码是初始化／设置代码，还是最终会落在热路径上的代码（例如处理服务中的每一个请求），往往就足够了。
* 这是将被许多应用使用的库代码吗？在这种情况下，很难判断它可能变得多么性能敏感。这正是遵循本文所描述的一些简单技巧变得尤为重要的地方。例如，如果你需要存储一个通常只有少量元素的向量，就使用 `absl::InlinedVector` 而不是 `std::vector`。这类技巧并不难遵循，也不会给系统增加任何非局部的复杂度。而如果你所编写的代码最终确实消耗了大量资源，它从一开始就会有更高的性能。并且在查看剖析结果时，也更容易找到下一个需要重点关注的地方。

当你在两个可能具有不同性能特征的选项之间做选择时，可以依靠 [信封背面估算 (back of the envelope calculations)](https://en.wikipedia.org/wiki/Back-of-the-envelope_calculation) 来做稍深入一些的分析。这类估算可以快速地给出不同方案性能的一个非常粗略的估计，其结果可用于在无需实现的情况下就淘汰掉一部分方案。

这样的估算大致可以这样进行：

1.  估算需要多少各种类型的底层操作，例如：磁盘寻道次数、网络往返次数、传输的字节数等。
2.  将每一种昂贵操作乘以其大致成本，然后把结果相加。
3.  上述过程给出的是系统以资源使用衡量的*成本 (cost)*。如果你关心的是延迟，并且系统具有一定的并发性，那么其中一些成本可能会相互重叠，你可能需要做稍微复杂一点的分析来估算延迟。

下面这张表是 [2007 年斯坦福大学一次演讲](https://static.googleusercontent.com/media/research.google.com/en//people/jeff/stanford-295-talk.pdf) 中一张表格的更新版本（2007 年演讲的视频已不复存在，但有一个 [涵盖部分相同内容的、相关的 2011 年斯坦福演讲视频](https://www.youtube.com/watch?v=modXC5IWTJI)）。它可能很有用，因为它列出了需要考虑的操作类型及其大致成本：

```shell
L1 缓存引用 (L1 cache reference)                        0.5 ns
L2 缓存引用 (L2 cache reference)                          3 ns
分支预测错误 (Branch mispredict)                          5 ns
互斥锁加锁/解锁（无竞争）(Mutex lock/unlock, uncontended) 15 ns
主内存引用 (Main memory reference)                        50 ns
用 Snappy 压缩 1K 字节                                 1,000 ns
从 SSD 读取 4KB                                       20,000 ns
同一数据中心内往返                                     50,000 ns
从内存顺序读取 1MB                                    64,000 ns
通过 100 Gbps 网络读取 1MB                           100,000 ns
从 SSD 读取 1MB                                   1,000,000 ns
磁盘寻道 (Disk seek)                              5,000,000 ns
从磁盘顺序读取 1MB                               10,000,000 ns
发送数据包 加州->荷兰->加州                       150,000,000 ns
```

上表包含了一些基本底层操作的粗略成本。你可能会发现，同时追踪与你的系统相关的一些更高层操作的估算成本也很有用。例如，你也许想知道对你的 SQL 数据库进行一次点读（point read）的大致成本、与某个云服务交互的延迟，或者渲染一个简单 HTML 页面所需的时间。如果你不知道不同操作的相关成本，你就无法做出像样的信封背面估算！

### 示例：对十亿个 4 字节数字进行快速排序所需的时间 (Example: Time to quicksort a billion 4 byte numbers)

作为一个粗略的近似，一个好的快速排序算法会对大小为 N 的数组进行 log(N) 趟遍历。在每一趟中，数组内容会从内存流入处理器缓存，分区代码会将每个元素与一个枢轴元素（pivot element）比较一次。让我们把主要成本加起来：

1.  内存带宽：该数组占用 4 GB（每个数字 4 字节乘以十亿个数字）。假设每个核心的内存带宽约为 16GB/s。这意味着每一趟大约需要 0.25s。N 约为 2^30，所以我们将进行约 30 趟，因此内存传输的总成本约为 7.5 秒。
2.  分支预测错误：我们总共会进行 N*log(N) 次比较，即约 300 亿次比较。假设其中一半（即 150 亿次）被错误预测。乘以每次预测错误 5 ns，得到预测错误成本为 75 秒。在此分析中，我们假设正确预测的分支是免费的。
3.  把前面的数字加起来，得到约 82.5 秒的估计值。

如有必要，我们可以进一步细化分析以考虑处理器缓存的影响。根据上面的分析，分支预测错误是主导成本，因此这一细化大概并不需要，但我们仍将其作为又一个示例放在这里。假设我们有一个 32MB 的 L3 缓存，并且从 L3 缓存传输数据到处理器的成本可忽略不计。L3 缓存可以容纳 2^23 个数字，因此最后 22 趟可以在驻留于 L3 缓存中的数据上进行操作（倒数第 23 趟把数据带入 L3 缓存，其余各趟在这些数据上操作）。这将内存传输成本从 7.5 秒（30 次内存传输）削减到 2.5 秒（以 16GB/s 传输 4GB，共 10 次内存传输）。

### 示例：生成一个包含 30 张图片缩略图的网页所需的时间 (Example: Time to generate a web page with 30 image thumbnails)

让我们比较两种可能的设计，其中原始图片存储在磁盘上，每张图片大小约为 1MB。

1.  串行读取这 30 张图片的内容，并为每一张生成一个缩略图。每次读取需要一次寻道 + 一次传输，寻道 5ms，传输 10ms，合计为 30 张图片乘以每张图片 15ms，即 450ms。
2.  并行读取，假设图片均匀分布在 K 个磁盘上。前面的资源使用估算仍然成立，但延迟将大致下降 K 倍（忽略方差；例如，我们有时会运气不好，某个磁盘上会有超过 1/K 的待读图片）。因此，如果我们运行在一个拥有数百个磁盘的分布式文件系统上，预期延迟将下降到约 15ms。
3.  让我们考虑一个变体，即所有图片都在单块 SSD 上。这将顺序读取的性能改变为每张图片 20µs + 1ms，总计约 30 ms。

## 测量 (Measurement)

前面一节给出了一些关于如何在编写代码时思考性能的技巧，而无需过多担心如何测量你的选择所带来的性能影响。然而，在你真正开始做改进之前，或者当你遇到涉及性能、简单性等各种因素之间的权衡时，你会想要测量或估算潜在的性能收益。能够有效地测量事物，是你在从事性能相关工作时首要想要拥有的工具。

顺带一提，值得指出的是，剖析你不熟悉的代码也可以是获得代码库整体结构及其运作方式的良好途径。审视程序动态调用图中那些高度参与的例程的源代码，能让你对运行代码时 " 发生了什么 " 有一个高层次的认识，进而建立起你在略微陌生的代码中进行性能改进改动的信心。

### 剖析工具和技巧 (Profiling tools and tips)

有许多有用的剖析工具可供使用。一个值得首先尝试的有用工具是 [pprof](https://github.com/google/pprof/blob/main/doc/README.md)，因为它能给出良好的高层次性能信息，并且无论是在本地还是对运行于生产环境中的代码都易于使用。如果你想获得更详细的性能洞察，也可以尝试 [perf](https://perf.wiki.kernel.org/index.php/Main_Page)。

剖析的一些技巧：

* 构建生产二进制文件时，带上适当的调试信息和优化标志。
* 如果可以，为你正在改进的代码编写一个 [微基准测试 (microbenchmark)][fast75]。微基准测试能缩短做性能改进时的周转时间、帮助验证性能改进的效果，并能帮助防止未来的性能回退。然而，微基准测试也可能存在一些 [陷阱][fast39]，使其无法代表整个系统的性能。用于编写微基准测试的有用库：[C++][cpp benchmarks] [Go][go benchmarks] [Java][jmh]。
* 使用基准测试库来 [输出性能计数器读数][fast53]，这样既能获得更好的精度，也能对程序行为获得更多洞察。
* 锁竞争（lock contention）常常会人为地降低 CPU 使用率。某些互斥锁实现提供了对锁竞争进行剖析的支持。
* 对于机器学习性能工作，使用 [ML 剖析器][xprof]。

> **Rust 备注（译者补充）**：上文提到的 pprof/perf/microbenchmark 工具在 Rust 生态中有对应选择——
> - 火焰图与 CPU 剖析：`cargo flamegraph`、`samply`；采集 pprof 格式数据可用 `pprof-rs`（`pprof` crate）。
> - 系统级细粒度剖析：仍可直接使用 `perf`（Linux），配合上述工具生成火焰图。
> - 微基准测试：使用 `criterion`（对应 C++/Go/Java 的基准库），它内置统计分析并能防止性能回退。
> - 构建生产二进制并保留调试信息：使用 `--release` 并在 `Cargo.toml` 的 profile 中设置 `debug = true`（例如 `[profile.release] debug = true`），以获得带符号的优化二进制。

### 当剖析结果平坦时该怎么办 (What to do when profiles are flat)

你经常会遇到 CPU 剖析结果平坦的情况（没有明显的、造成缓慢的大贡献者）。这种情况通常发生在所有 " 低垂的果实 " 都已被摘取之后。如果你发现自己身处此种境地，以下是一些可供考虑的技巧：

* 不要低估许多小优化的价值！在某个子系统中做出二十个各自 1% 的独立改进往往是完全可行的，合起来就意味着相当可观的整体提升（此类工作往往依赖于拥有稳定且高质量的微基准测试）。这类改动的一些示例，见 [展示多种技术的改动](#cls-that-demonstrate-multiple-techniques) 一节。
* 寻找靠近调用栈顶部的循环（CPU 剖析的火焰图视图在此会很有帮助）。有可能，这个循环或它所调用的代码可以被重构得更高效。有一段代码最初通过遍历输入的节点和边、以增量方式构建一个复杂的图结构，后来被改为通过把整个输入一次性传入来一次性构建该图结构。这消除了初始代码中每条边都会发生的一堆内部检查。
* 退后一步，从更高的调用栈层面寻找结构性的改动，而不是专注于微优化。在做这件事时，[算法改进](#algorithmic-improvements) 一节列出的技术会很有用。
* 寻找过于通用的代码。用一个定制的或更底层的实现来替换它。例如，如果某个应用反复使用正则表达式匹配，而其实一个简单的前缀匹配就已足够，那就考虑放弃使用正则表达式。
* 尝试减少分配（allocation）的数量：[获取一份分配剖析][profile sources]，并从分配数量最高的贡献者开始逐个击破。这会带来两个效果：(1) 它会直接减少花在分配器（对于有 GC 的语言还包括垃圾回收器）上的时间；(2) 通常还会减少缓存未命中，因为在一个长时间运行、使用 tcmalloc 的程序中，每次分配往往会落到不同的缓存行上。
* 收集其他类型的剖析，尤其是基于硬件性能计数器的剖析。这类剖析可能会指出那些遭遇高缓存未命中率的函数。[剖析工具和技巧](#profiling-tools-and-tips) 一节中描述的技术会很有帮助。

## API 设计考量 (API Considerations)

下面建议的一些技术需要改动数据结构和函数签名，这可能会对调用方造成干扰。应尽量组织代码，使这些性能改进能够在封装边界内部完成，而不影响公共接口。如果你的 [模块足够"深"](https://web.stanford.edu/~ouster/cgi-bin/book.php)（即通过窄接口访问丰富功能），这会更容易做到。

被广泛使用的 API 常常面临添加新特性的巨大压力。添加新特性时要谨慎，因为它们会约束未来的实现，并给那些不需要该特性的用户带来不必要的成本。例如，许多 C++ 标准库容器承诺迭代器稳定性，这在典型实现中会显著增加分配次数，即便许多用户并不需要指针稳定性。

下面列出一些具体技术。请仔细权衡性能收益与此类改动引入的 API 易用性问题。

### 批量 API (Bulk APIs)

提供批量操作，以减少昂贵的 API 边界穿越，或利用算法层面的改进。

- **添加批量 `MemoryManager::LookupMany` 接口** — 除了增加批量接口外，还简化了新批量变体的签名：结果表明调用方只需要知道是否所有 key 都被找到，因此可以返回 `bool` 而非 `Status` 对象。

  Rust 备注：批量接口用切片传入 key、切片写回结果，返回 `bool` 表示是否全部命中：

  ```rust
  fn lookup_many(&self, keys: &[LookupKey], tensors: &mut [Tensor]) -> bool;
  ```

- **添加批量 `ObjectStore::DeleteRefs` API 以摊销加锁开销** — 原先每个 ref 单独调用 `DeleteRef` 都要加锁；批量版本一次加锁后循环删除所有 ref，任一错误则返回非 OK。调用方由逐个删除句柄改为整批传入 `DeleteRefs(handles)`。

  Rust 备注：一次锁定、批量删除：

  ```rust
  fn delete_refs(&self, refs: &[Ref]) -> Result<(), Error> {
      let _guard = self.mu.lock().unwrap();
      let mut result = Ok(());
      for r in refs {
          if let Err(e) = self.delete_ref_locked(*r) { result = Err(e); }
      }
      result
  }
  ```

- **使用 [Floyd 建堆法](https://en.wikipedia.org/wiki/Heapsort#Variations) 进行高效初始化** — 批量初始化堆可在 O(N) 时间内完成，而逐个添加元素并在每次添加后维护堆性质则需要 O(N lg(N)) 时间。

  Rust 备注：`BinaryHeap::from(vec)` 走的是 Floyd 建堆，O(N)；而循环 `push` 逐个插入是 O(N log N)：

  ```rust
  let heap = BinaryHeap::from(vec);          // O(N)，Floyd 建堆
  // 对比：
  let mut heap = BinaryHeap::new();
  for x in vec { heap.push(x); }             // O(N log N)
  ```

有时很难让调用方直接改用新的批量 API。这种情况下，可以在内部使用批量 API 并缓存结果，供未来的非批量调用使用：

- **缓存 block 解码结果供未来调用使用** — 每次查找都需要解码一整块 K 个条目。将解码后的条目存入缓存，未来查找时先查缓存；缓存命中则直接返回，未命中才解码整块并填充缓存。

### 视图类型 (View Types)

对函数参数优先使用视图类型（如 `std::string_view`、`std::Span<T>`、`absl::FunctionRef<R(Args…)>`），除非要转移数据所有权。这些类型减少拷贝，并允许调用方自选容器类型（例如一个调用方用 `std::vector`，另一个用 `absl::InlinedVector`）。

**Rust 备注**：Rust 中对应的视图类型：

- `std::string_view` → `&str`
- `Span<T>` → `&[T]`
- `FunctionRef<R(Args…)>` → `&dyn Fn(Args) -> R` 或泛型 `impl Fn(Args) -> R`

```rust
fn process(name: &str, data: &[u8], cb: impl Fn(u32) -> u32) { /* ... */ }
```

### 预分配 / 预计算参数 (Pre-allocated/pre-computed arguments)

对于频繁调用的例程，有时允许上层调用方传入它们自己拥有的数据结构、或被调例程所需而客户端已有的信息，会很有用。这可以避免底层例程被迫分配自己的临时数据结构，或重新计算已经可用的信息。

- **添加 `RPC_Stats::RecordRPC` 变体，允许客户端传入已有的 `WallTime` 值** — 原签名为 `RecordRPC(name, m)`，新签名增加 `WallTime now` 参数。调用方 `clientchannel.cc` 本已通过 `WallTime_Now()` 得到 `now`，直接把它传进去，避免例程内部重复取时间。

  Rust 备注：把已有的时间戳作为参数传入，避免函数内部重新取时：

  ```rust
  fn record_rpc(name: &Name, m: &Measurement, now: Instant) { /* ... */ }
  let now = Instant::now();
  record_rpc(&stats_name, &m, now);
  ```

### 线程兼容 vs. 线程安全类型 (Thread-compatible vs. Thread-safe types)

一个类型可以是线程兼容的（由外部同步）或线程安全的（内部自行同步）。大多数通用类型应当是线程兼容的。这样，不需要线程安全的调用方就不必为其付出代价。

- **将某个类改为线程兼容，因为调用方已经做了同步** — `HitlessTransferPhase::get()` 和 `AllowAllocate()` 原本各自 `MonitoredMutexLock` 加锁再读 `phase_`；由于调用方已同步，直接去掉锁，改为裸读 `return phase_;`。

  Rust 备注：线程兼容 = 普通结构体，同步交给外部 `Mutex`；内部同步 = 用 `dashmap` 等内部加锁容器。

  ```rust
  // 线程兼容：由外部 Mutex 保护
  struct HitlessTransferPhase { phase: TransferPhase }
  let shared = Mutex::new(HitlessTransferPhase { phase });

  // 内部同步：类型自己负责加锁
  use dashmap::DashMap;
  let map: DashMap<K, V> = DashMap::new();
  ```

然而，如果一个类型的典型用法都需要同步，则更倾向于把同步移到类型内部。这样便可按需调整同步机制以提升性能（例如分片以降低争用），而不影响调用方。

## 算法改进 (Algorithmic improvements)

最关键的性能改进机会来自算法层面的改进，例如把 O(N²) 算法变为 O(N lg(N)) 或 O(N)、避免潜在的指数级行为等。这类机会在稳定代码中很少见，但在编写新代码时值得关注。下面是几个对既有代码进行此类改进的例子：

- **以逆后序（reverse post-order）向环检测结构添加节点** — 之前是一次一个地向环检测数据结构添加图节点和边，每条边都需要昂贵的处理。现在改为以逆后序一次性添加整个图，使环检测变得简单：对无环图做 DFS 会以逆拓扑序离开节点，因此离开节点时赋予递减的 rank；随后一次性添加所有边，边的 `rank[src] >= rank[dst]` 即表示有环。新增 `InitFrom` 接口，`MergeGraph::Init()` 由逐边 `InsertEdge` 改为一次 `InitFrom(graph)`。

- **用更好的算法替换 mutex 实现内置的死锁检测系统** — 新算法约为旧算法的 ~50 倍快，且能无问题地扩展到数百万个 mutex（旧算法依赖 2K 上限来避免性能悬崖）。新代码基于论文：David J. Pearce、Paul H. J. Kelly，《A dynamic topological sort algorithm for directed acyclic graphs》，Journal of Experimental Algorithmics (JEA)，第 11 卷，2006 年，Article No. 1.7。新算法占用 O(|V|+|E|) 空间（而非旧算法所需的 O(|V|²) 比特）；由于加锁顺序图非常稀疏，空间省得多。算法也很简单，核心约 100 行 C++。因为现在能扩展到多得多的 Mutex 数量，得以放宽人为的 2K 上限，这在真实程序中暴露出了若干潜在死锁。基准（DEBUG 模式，参数为被跟踪节点数）：在旧算法默认的 2K 上限处，新算法每次 InsertEdge 仅需 0.5 微秒，而旧算法需要 22 微秒；新算法可轻松扩展到大得多的图，而旧算法很快就崩溃。

- **用哈希表（O(1) 查找）替换 `IntervalMap`（O(lg N) 查找）** — 最初用 `IntervalMap` 是因为它看起来是支持合并相邻块的合适数据结构，但哈希表已足够：相邻块可通过一次哈希表查找找到（哈希表条目额外记录前一个块的长度，以便找到前驱块合并）。此改动（连同该 CL 中的其他改动）使 `tpu::BestFitAllocator` 的性能提升约 ~4X。

- **用哈希表查找（O(N)）替换有序列表求交（O(N log N)）** — 旧代码为检测两个节点是否共享公共源，会以有序方式取出每个节点的源再做有序求交；新代码把一个节点的源放入哈希表，然后遍历另一节点的源逐个查表。基准：`BM_CompileLarge` 从 28.5s 降到 22.4s，即 **-21.61%**（p=0.008，n=5+5）。

- **实现好的哈希函数，使操作从 O(N) 变为 O(1)** — 原 `LocationHash` 只对 `key->address()` 求哈希，导致大量 `Location` 落到同一桶而退化。新增 `HashLocation`，把 `dynamic/is_any/has_shardmap/has_sharding` 等布尔特征编码进单个值，再混入 `shardmap`、`sharding` 的数值以及 `address`、`lb_policy` 字符串（使用 `MurmurCat` 累积），从而得到分布良好的哈希，使查找回到 O(1)。

## 更好的内存表示 (Better memory representation)

仔细考量重要数据结构的内存占用与缓存占用，往往能带来巨大的收益。下面的数据结构着眼于以触碰更少缓存行的方式支持常见操作。在此下功夫可以：(a) 避免昂贵的缓存未命中；(b) 减少内存总线流量，从而既加速当前程序，也加速同一台机器上运行的其他一切。它们依赖一些通用技巧，你在设计自己的数据结构时可能会发现它们很有用。

### 紧凑数据结构 (Compact data structures)

对于会被频繁访问、或占应用内存使用很大比例的数据，采用紧凑表示。紧凑表示能显著减少内存使用，并通过触碰更少缓存行、降低内存总线带宽占用来提升性能。但要小心缓存行争用（cache-line contention）。

### 内存布局 (Memory layout)

对于具有较大内存或缓存占用的类型，仔细考量其内存布局。

- 重排字段，减少不同对齐要求字段之间的填充（padding）。
- 当存储的数据能放进更小的数值类型时，使用更小的数值类型。
- 枚举值如果不加注意，有时会占用整个机器字。考虑使用更小的表示，例如用 `enum class OpType : uint8_t { … }` 而不是 `enum class OpType { … }`。
- 把经常一起访问的字段安排得彼此靠近——这会减少常见操作触碰的缓存行数。
- 把热的只读字段与热的可变字段分开摆放，这样对可变字段的写入就不会导致只读字段从邻近缓存中被逐出（这是单线程下的缓存逐出问题，即写入不应把只读字段挤出附近缓存，而非多线程伪共享）。
- 移动冷数据，使其不与热数据相邻：可将冷数据放在结构体末尾、置于一层间接引用之后、或放入单独的数组。
- 考虑通过位级/字节级编码把数据打包进更少的字节。这可能很复杂，因此仅当相关数据被封装在经过充分测试的模块内、且整体内存使用的削减很显著时才这么做。此外，要提防副作用，如频繁使用数据的欠对齐（under-alignment），或访问打包表示时代码更昂贵。请用基准测试（benchmarks）来验证这类改动。

**Rust 对应**：Rust 结构体默认会重排字段以最小化填充；若需固定 C 布局用 `#[repr(C)]`。用更小的整型（`u8`/`u16`/`u32`）替代默认 `i32`/`i64`，枚举可用 `#[repr(u8)]` 控制判别式宽度。位打包可用 `bitflags`、`bitfield` 等 crate，同样需用 `criterion` 基准验证。

### 索引代替指针 (Indices instead of pointers)

在现代 64 位机器上，指针占 64 位。如果数据结构富含指针，`T*` 间接引用很容易吃掉大量内存。改用整数索引指向数组 `T[]` 或其他数据结构：不仅引用更小（若索引数量小到能放进 32 位或更少），而且所有 `T[]` 元素的存储是连续的，通常带来更好的缓存局部性。

**Rust 对应**：用 `Vec<T>` + `u32` 索引替代 `Box<T>`/`&T`。对图等结构，`id-arena` 或 `slotmap` 是常用方案；这种做法还能绕开借用检查器（borrow checker）对图结构的限制，因为索引不是引用。

### 批量存储 (Batched storage)

避免为每个存储元素单独分配一个对象的数据结构（如 C++ 的 `std::map`、`std::unordered_map`）。改用采用分块或扁平表示、把多个元素在内存中紧邻存放的类型（如 C++ 的 `std::vector`、`absl::flat_hash_{map,set}`）。这类类型往往有好得多的缓存行为，并且分配器开销更小。

一个有用的技巧是把元素划分成块，每块可容纳固定数量的元素。此技巧能显著减少数据结构的缓存占用，同时保持良好的渐近行为。

对某些数据结构，单个块就足以容纳所有元素（如字符串和向量）。其他类型（如 `absl::flat_hash_map`）也使用这一技巧。

**Rust 对应**：标准库的 `HashMap`（基于 hashbrown）本身就是扁平/开放寻址实现，元素存放于连续存储中，无需每元素单独分配；直接使用 `Vec`、`HashMap` 即可获得良好的缓存行为。

### 内联存储 (Inlined storage)

一些容器类型针对存储少量元素做了优化。它们在顶层为少量元素预留空间，元素数量少时完全避免分配。当这类类型的实例被频繁构造（如作为频繁执行代码中的栈变量）、或同时有很多实例存活时，这非常有帮助。如果一个容器通常只含少量元素，考虑使用内联存储类型之一，如 InlinedVector。

注意：如果 `sizeof(T)` 很大，内联存储容器可能不是最佳选择，因为内联后备存储会很大。

**Rust 对应**：用 `smallvec::SmallVec` 或 `arrayvec::ArrayVec`，在元素少时把数据存在栈上避免堆分配。

### 不必要的嵌套映射 (Unnecessarily nested maps)

有时嵌套的映射数据结构可以用带复合键的单层映射替代，这能显著降低查找和插入的成本。

- 示例（graph_splitter.cc）：把 `btree<a, btree<b, c>>` 转换为 `btree<pair<a,b>, c>`，从而减少分配并改善缓存占用。原来是 `absl::btree_map<std::string, absl::btree_map<std::string, OpDef>> ops;`，改为以 `{package_name, op_name}` 组成的 `std::pair` 为键、映射到 `const OpDef*` 的单层 btree。

注意：如果第一层映射键很大，保留嵌套映射可能更好：

- 示例：改用嵌套映射带来微基准 76% 的性能提升。原先是单层哈希表，键由一个（字符串）路径和一些数值子键组成，平均每个路径出现在约 1000 个键中。将其拆成两层：第一层以路径为键，每个第二层哈希表只保存该特定路径下 " 子键到数据 " 的映射。这把存储路径的内存使用减少了 1000 倍，也加速了同一路径下许多子键被一起访问的场景。

**Rust 对应**：用复合键单层 `HashMap<(A, B), C>` 替代 `HashMap<A, HashMap<B, C>>`。

### 竞技场分配 (Arenas)

竞技场（arena）能帮助降低内存分配成本，但它们还有额外的好处：把独立分配的项目彼此紧邻打包，通常落在更少的缓存行内，并消除大部分析构成本。它们对含有许多子对象的复杂数据结构最为有效。考虑为竞技场提供一个合适的初始大小（initial size），这有助于减少分配。

注意：竞技场很容易被误用——把太多短生命周期对象放进一个长生命周期的竞技场，会不必要地膨胀内存占用。

**Rust 对应**：`bumpalo::Bump` 不会对其中内容运行 `Drop`（这正是 " 消除析构成本 " 的来源），因此不要把持有需要清理资源的 `Drop` 类型放进去；而 `typed-arena::Arena<T>` 在竞技场本身被 drop 时，会对其中内容运行 `Drop`。根据是否需要析构语义选择合适的 crate。

### 数组代替映射 (Arrays instead of maps)

如果映射的定义域可用一个小整数表示、或是一个枚举、或者映射元素极少，那么该映射有时可以用某种数组或向量替代。

- 示例（rtp_controller.h）：用数组代替 flat_map。原来是 `const gtl::flat_map<int, int> payload_type_to_clock_frequency_;`，改为以 `payload_type` 索引、大小为 128 的普通数组结构 `struct PayloadTypeToClockRateMap { int map[128]; };`（用简单数组实现的映射，映射到该 payload type 的时钟频率，无则为 0）。

**Rust 对应**：用 `[T; N]` 并以枚举（`as usize`）作为下标索引。

### 位向量代替集合 (Bit vectors instead of sets)

如果集合的定义域可用一个小整数表示，该集合可以用位向量替代（InlinedBitVector 通常是不错的选择）。在这些表示上，集合运算也可以用按位布尔运算很高效地完成（OR 求并集，AND 求交集，等等）。

- 示例（Spanner 放置系统，zone_set.h）：用每个 zone 一个比特的位向量替代 `dense_hash_set<ZoneId>`。原来 `ZoneSet` 继承自 `dense_hash_set<ZoneId>`，`Contains` 用 `count(zone) > 0`；改为内部用 `util::bitmap::InlinedBitVector<256> b_`（外加 `int size_` 记录已插入 zone 数），`ContainsZone` 判断 `zone < b_.size() && b_.get_bit(zone)`。基准结果（AMD Opteron 4 核，dL1:64KB dL2:1024KB）显示约 30% 的提升：BM_Evaluate/1 从 960ns→676ns（+29.6%）、/2 1661→1138（+31.5%）、/3 2305→1640（+28.9%）、/4 3053→2135（+30.1%）、/5 3780→2665（+29.5%）、/10 7819→5739（+26.6%）、/20 17922→12338（+31.2%）、/40 36836→26430（+28.2%）。
- 示例（hlo_computation.h）：用位矩阵（bit matrix）跟踪操作数之间的可达性属性，替代哈希表。原来 `TransitiveOperandMap = std::unordered_map<const HloInstruction*, std::unordered_set<const HloInstruction*>>`；改为 `ReachabilityMap`，内部用 `FlatMap<const HloInstruction*, int> ids_` 做从指针到编号的稠密 id 分配，再用 `Bitmap matrix_`，其中 `matrix_(a,b)` 为真当且仅当 b 从 a 可达。

**Rust 对应**：用 `fixedbitset::FixedBitSet` 或 `bitvec` 表示位向量/位矩阵。

## 减少分配 (Reduce allocations)

内存分配带来额外成本：

1. 增加花在分配器中的时间。
2. 新分配的对象可能需要昂贵的初始化，并在不再需要时有时需要对应的昂贵析构。
3. 每次分配往往落在新的缓存行上，因此散布在许多独立分配上的数据，其缓存占用会大于散布在较少分配上的数据。

垃圾回收运行时有时会通过把连续分配顺序放置在内存中来消除第 3 个问题。

### 避免不必要的分配 (Avoid unnecessary allocations)

- 示例（memory_manager.cc）：减少分配使基准吞吐量提升 21%。原 `LiveTensor` 构造函数在 `dinfo` 为空时用 `std::make_shared<DeviceInfo>()` 每次都分配；改为复用一个静态的空默认对象：定义 `empty_device_info()` 返回一个静态构造的 `std::shared_ptr<DeviceInfo>`，构造函数在 `dinfo` 存在时 `std::move(dinfo)`，否则赋值为 `empty_device_info()`，从而避免反复分配。
- 示例（embedding_executor_8bit.cc）：能用静态分配的零向量时就用它，而不是分配一个向量再填零。原来每次 `new tensorflow::quint8[max_embedding_width]` 并 `memset` 清零；改为定义静态数组 `static tensorflow::quint8 static_zero_data[kTypicalMaxEmbedding]`（`kTypicalMaxEmbedding = 256`，足以覆盖大多数 embedding 宽度，全零）。当 `max_embedding_width <= ARRAYSIZE(static_zero_data)` 时直接指向静态数组、无需分配；否则才回退到动态分配并清零。

此外，当对象生命周期由作用域限定时，优先栈分配而非堆分配（但要小心大对象的栈帧大小）。

**Rust 对应**：用 `static`/`OnceLock`/`lazy_static` 缓存复用的默认对象；优先栈上值而非 `Box`/`Vec` 堆分配。

### 调整大小或预留容器容量 (Resize or reserve containers)

当一个向量（或某些其他容器类型）的最大或预期最大大小事先已知时，预先设置容器后备存储的大小（如 C++ 中用 `resize` 或 `reserve`）。

- 示例（indexblockdecoder.cc）：预先设定向量大小再填充，而不是做 N 次 push_back。原来在循环里对 `docs_` 反复 `push_back`；改为先 `docs_.resize(ndocs)`，取原始指针 `DocId* docptr = &docs_[0]` 后在循环中直接写入并递增指针。

注意（两条）：
1. 不要用 `resize`/`reserve` 一次只增长一个元素，那可能导致二次方（quadratic）行为。
2. 如果元素构造很昂贵，优先做一次初始 `reserve` 再跟若干 `push_back`/`emplace_back`，而不是初始 `resize`——因为 `resize` 会让构造函数调用次数翻倍。

**Rust 对应**：用 `Vec::with_capacity` / `reserve`；同理不要在循环里逐个 `reserve(1)`。`resize` 会为新元素调用构造/克隆，若元素构造昂贵，优先 `with_capacity` + `push`。

### 尽可能避免拷贝 (Avoid copying when possible)

- 尽可能优先移动（move）而非拷贝数据结构。
- 如果生命周期不成问题，在瞬态数据结构中存储指针或索引，而不是对象的拷贝。例如，若用一个局部 map 从传入的 proto 列表中挑选一组 proto，可让 map 只存指向传入 proto 的指针，而不是拷贝可能深层嵌套的数据。另一个常见例子是排序索引向量而不是直接排序大对象向量，因为后者会带来显著的拷贝/移动成本。

- 示例：通过 gRPC 接收张量时避免一次额外拷贝。发送约 400KB 张量的基准提速约 10-15%：BM_RPC/30/98k_mean 从 Time 148764691ns / CPU 1369998944ns 降到 Time 131595940ns / CPU 1216998084ns（1000 次迭代）。
- 示例（index.cc）：移动而非拷贝大的 options 结构。`Create(opts)` 改为 `Create(std::move(opts))`。
- 示例（encoded-vector-hits.h）：用 `std::sort` 代替 `std::stable_sort`，避免稳定排序实现内部的一次拷贝。原来 `std::stable_sort(hits_…, gtl::OrderByField(&HitWithPayloadOffset::docid))`；改为给 `HitWithPayloadOffset` 定义 `operator<`（先比 `docid`，相等再比 `first_payload_offset`）后调用 `std::sort(hits_.begin(), hits_.end())`。

**Rust 对应**：Rust 默认按移动语义传值，所以 " 优先移动 " ≈ " 避免不必要的 `.clone()`"。排序时优先 `sort_unstable`（不稳定，无内部额外分配）而非 `sort`（稳定），除非确实需要稳定性。

### 复用临时对象 (Reuse temporary objects)

在循环内部声明的容器或对象，会在每次循环迭代时被重新创建，这会导致昂贵的构造、析构和扩容。把声明提升（hoist）到循环外面即可复用，能带来显著的性能提升。（由于语言语义或无法保证程序等价，编译器往往无法自行做这种提升。）

- 示例（autofdo_profile_utils.h）：把变量定义提升到循环迭代之外。原来在 `while` 循环内声明 `T profile;`；改为在循环外声明 `T profile;`，循环内复用它解析。
- 示例（stats-router.cc）：在循环外定义 protobuf 变量，使其分配的存储能跨迭代复用。原来在 `for` 循环内声明 `ResourceRecord record;`；改为在循环外声明，循环内先 `record.Clear();` 再复用。
- 示例（program_rep.cc）：反复序列化到同一个 `std::string`。原来 `DeterministicSerialization` 每次内部 `std::string result;`；改为 `DeterministicSerializationTo(m, std::string* scratch)`，先 `scratch->clear()` 复用外部传入的缓冲，返回 `absl::string_view(*scratch)`。

注意：protobuf、string、vector、容器等往往会增长到曾经存入的最大值的大小。因此定期重建它们（例如每使用 N 次之后重建）有助于减少内存需求和重新初始化成本。

**Rust 对应**：把 `String`/`Vec` 等缓冲提升到循环外，每次迭代用 `.clear()`（保留容量）后复用；但由于容量会保持在历史最大值，可在使用 N 次后重新构造以回收内存。

## 避免不必要的工作

或许提升性能最有效的一类手段，就是避免那些本就不必做的工作。它有很多种形式，包括：为常见情形创建专门的代码路径以绕开更通用、更昂贵的计算；预计算；把工作推迟到真正需要时再做；把工作提升到执行频率更低的代码片段中；以及其他类似的做法。下面给出这一通用思路的大量示例，并归入几个有代表性的类别。

### 为常见情形提供快速路径

代码常常被写成覆盖所有情形，但其中某些子集比其他情形要简单得多、也常见得多。例如 `vector::push_back` 通常有足够空间容纳新元素，但也包含在空间不足时对底层存储进行扩容的代码。对代码结构稍加注意，就能让常见的简单情形更快，同时又不会显著损害不常见情形的性能。

- **让快速路径覆盖更多常见情形。**（utf8statetable.cc）为该例程增加对末尾单个 ASCII 字节的处理，而不再只处理 4 字节的整数倍。这样一来，对于长度例如为 5 字节的全 ASCII 字符串，就不必再调用较慢的通用例程。原实现只用 `UNALIGNED_LOAD32` 每次跳过 8 个 ASCII 字节，然后立即调用通用 `UTF8GenericScan`；新实现在跳过 8 字节块之后，再用 `while (src < src_limit && Is7BitAscii(*src)) src++;` 逐字节吃掉剩余的 ASCII，仅当 `src < src_limit` 时才调用通用扫描。

- **为 InlinedVector 提供更简单的快速路径。**（inlined_vector.h）原 `Storage<T,N,A>::Resize` 先建立 `AllocationTransaction`/`ConstructionTransaction` 以及三个 `absl::Span`（construct_loop、move_construct_loop、destroy_loop）再分支。新实现先把 `storage_view.data`、`storage_view.size` 复制成局部变量 `base`、`size`，然后直接按 `new_size <= size`（销毁多余旧元素）、`new_size <= capacity`（就地构造新元素）、否则（扩容）三路分支，去掉了事务与 Span 的开销。

- **为初始化 1 维到 4 维张量提供快速路径。**（tensor_shape.cc）原 `TensorShapeBase` 构造函数对每一维都循环调用 `AddDim(SubtleMustCopy(s))`。新实现拆出 `InitDims`：先用常量 `kMaxSmall = 0xd744`（并静态断言 `kMaxSmall^4 <= kint64max`，保证下面的 4 路乘法不溢出）检查是否所有维度都不超过它；若全部适配 16 位，则对维数为 1、2、3、4 的情形分别用 `switch` 走快速路径（例如 case 2 直接算 `size0 * size1`）；否则退回原来的逐维 `AddDim` 慢路径。

- **让 varint 解析器的快速路径只覆盖 1 字节情形，而非同时覆盖 1 字节和 2 字节情形。**（parse_context.h / parse_context.cc）缩小（被内联的）快速路径体积可以减少代码大小和 icache 压力，从而提升性能。原 `VarintParse` 内联处理 1 字节和 2 字节两种情形后才调用 `VarintParseSlow`；新实现只内联 1 字节情形，其余一律交给 `VarintParseSlow`。相应地，`VarintParseSlow32`/`VarintParseSlow64` 的循环起点由 `i = 2` 改回 `i = 1`。

- **若未发生任何错误，则跳过 RPC_Stats_Measurement 相加中的大量工作。**（rpc-stats.h / rpc-stats.cc）为 `RPC_Stats_Measurement` 增加私有的 `bool any_errors_set`，并把 `errors[]` 数组改为私有、通过 `set_errors` 写入时置位标志。`operator+=` 中只有当 `x.any_errors_set` 为真时，才遍历 `RPC::NUM_ERRORS` 逐项累加错误计数。

- **对字符串首字节做数组查找，从而常常避免对整串做指纹计算。**（soft-tokens-helper.h / .cc）由于软 token 大多与标点相关，新增数组 `bool filter_[256]`：`filter_[i]` 为真当且仅当有某个软 token 以字节值 `i` 开头。内联的 `IsSoftToken` 先看首字节，若 `filter_[first_char]` 为假就立即返回 false；否则才调用 `IsSoftTokenFallback`，后者仍用 `Fingerprint(token)` 在 `soft_tokens_` 中查找。

**Rust 备注**：快速路径 + 慢路径的惯用写法是把慢路径拆成单独函数并标注 `#[cold]` 与 `#[inline(never)]`，让编译器把它移出内联体：

```rust
#[inline]
fn varint_parse(p: &[u8], out: &mut u32) -> usize {
    let b0 = p[0];
    if b0 & 0x80 == 0 { *out = b0 as u32; return 1; }
    varint_parse_slow(p, b0 as u32, out)
}
#[cold]
#[inline(never)]
fn varint_parse_slow(p: &[u8], res: u32, out: &mut u32) -> usize { /* ... */ }
```

首字节过滤表可用 `const FILTER: [bool; 256] = { /* … */ };` 在编译期构造。全 ASCII 快扫可用 `slice::iter().position(|&b| b & 0x80 != 0)` 或按块处理。

### 一次性预计算昂贵信息

- **预计算 TensorFlow 图执行节点的一个属性，从而能快速排除某些不寻常情形。**（executor.cc）原 `NodeItem` 用若干 `bool` 字段并在执行时反复调用 `IsEnter(node)`/`IsExit(node)`/`IsNextIteration(node)` 判断。新实现把这些标志改为位域（`bool kernel_is_expensive : 1;` 等），并新增预计算的 `is_enter`、`is_exit`、`is_control_trigger`、`is_sink`，以及 `is_enter_exit_or_next_iter`（当 `IsEnter||IsExit||IsNextIteration` 时为真）。执行时先判断 `!item->is_enter_exit_or_next_iter` 走快速路径。

- **预计算 256 元素数组，并在 trigram 初始化时使用。**（byte_trigram_classifier.cc）原 `VerifyModel` 在双重循环中对每个 trigram、每个类别调用 `Prob(trigram_probs_[id].log_probs[cls])`。新实现先 `CHECK_EQ(sizeof(ByteLogProbT), 1)`，然后预填 `ProbT fast_prob[256]`（`fast_prob[b] = Prob(static_cast<ByteLogProbT>(b))`），循环中改成查表 `fast_prob[…]`。

一般性建议：应在模块边界处检查非法输入，而不是在模块内部反复检查。

> **Rust 备注**：256 元素查表用 `const` 数组即可编译期生成（或用 `std::array::from_fn`/`LazyLock` 运行期填充）。位域可用 `bitflags` crate 或手工位操作表达。

### 把昂贵的计算移出循环

- **把边界计算移出循环。**（literal_linearizer.cc）原循环条件每次迭代都调用 `src_shape.dimensions(dimension_numbers.front())`。新实现在循环外先取 `int64 dim_front = src_shape.dimensions(dimension_numbers.front());`，并把 `src_buffer.data()`、`dst_buffer.data()` 提到局部变量 `src_buffer_data`/`dst_buffer_data`，循环条件改为 `i < dim_front`。

> **Rust 备注**：把循环不变量提到循环外的局部变量即可（编译器有时也能自动做到，但显式提取更保险，也能减少 bounds check）。

### 推迟昂贵的计算

- **把 GetSubSharding 调用推迟到需要时再做，将 43 秒 CPU 时间降到 2 秒。**（sharding_propagation.cc）原代码无条件先算 `alternative_sub_sharding = user.sharding().GetSubSharding(user.shape(), {i})`，再判断 `user.operand(i) == &instruction`。新实现把判断前置：仅当 `user.operand(i) == &instruction` 这个操作数确实相关时，才去计算相对昂贵的 `GetSubSharding`。

- **不要急切地更新统计信息，而应按需计算。**在极其频繁的分配/释放调用上不更新统计；改为在调用频率低得多的 `Stats()` 方法被触发时，按需计算统计。

- **在 Google 的 Web 服务器中，查询处理只预分配 10 个节点而非 200 个。**（querytree.h）一个简单改动，把 querynode 池的初始大小 `kInitParseTreeSize` 从 `200` 改为 `10`，使 Web 服务器 CPU 使用量降低了 7.5%。

- **改变搜索顺序，带来 19% 的吞吐提升。**一个早期（约 2000 年）的搜索系统分两层：一层是全文索引，另一层只含标题和锚文本词项的索引。过去先搜较小的标题/锚文本层。反直觉的是，我们发现先搜较大的全文索引层反而更省：因为一旦扫到全文层末尾，就可以完全跳过对标题/锚文本层（它是全文层的子集）的搜索。这种情况相当常见，从而降低了处理一次查询的平均磁盘寻道次数。背景可参见《The Anatomy of a Large-Scale Hypertextual Web Search Engine》中关于标题与锚文本处理的讨论。

> **Rust 备注**：延迟计算可用 `std::sync::LazyLock`（全局/静态）或 `once_cell::OnceCell` / `Cell<Option<T>>`（实例级）来惰性求值，仅在首次访问时计算一次。

### 特化代码

某个对性能敏感的调用点，未必需要通用库所提供的全部通用性。在这种情况下，如果能带来性能提升，可以考虑编写专门的特化代码，而不是调用通用代码。

- **Histogram 类的自定义打印代码比 sprintf 快 4 倍。**（histogram_export.cc）这段代码对性能敏感，因为监控系统从各服务器收集统计时会调用它。原 `PopulateBuckets` 在循环中反复用 `StringPrintf("%.12g" / "%.0f-%.0f" / "%.12g_%.12g", …)` 格式化并 `EscapeMapKey`、`StrCat` 拼前缀。新实现引入特化的 `FormatNumber(format, need_escape, val, scratch)`：只支持 `"%.0f"`/`"%.12g"` 两种格式（`DCHECK` 校验），对整数值直接 `StrAppend(int32)`，对无穷大直接返回 `"inf"`，其余才走 `StringAppendF` 并按需转义；循环外预先算好 `full_key_prefix`，循环内用 `key.resize(…)` 复用缓冲区并把 `cumul_counts`/`discrete` 提为循环外常量。

- **为 VLOG(1)、VLOG(2)…… 增加特化，以提高速度并减小代码体积。**（vlog_is_on.h / .cc）`VLOG` 是代码库中被大量使用的宏。由于调用点的日志级别几乎总是编译期常量（如 `VLOG(1) << …`），此改动避免在几乎每个调用点传递一个额外的整型常量，从而节省代码空间。新实现在 `IsEnabled` 中，若 `__builtin_constant_p(level)` 成立，则按 level 分派到无内联的 `SlowIsEnabled0..5`；否则回退到 `SlowIsEnabled(stale_v, level)`。这些 `SlowIsEnabledN` 都是 `ABSL_ATTRIBUTE_NOINLINE`，实现上转调 `SlowIsEnabled(stale_v, N)`。

- **在可能时，用简单的前缀匹配替代 RE2 调用。**（read_matcher.cc）为 `MatchItemType` 枚举新增 `MATCH_TYPE_PREFIX`（用于正则 `".*"` 的特例）。构造匹配项时，先 `term.NonMetaPrefix().CopyToString(&p->prefix)`；如果 `term.RegexpSuffix() == ".*"`，说明该正则匹配任意内容，可绕过 `RE2::FullMatch`，将类型设为 `MATCH_TYPE_PREFIX`；否则才设为 `MATCH_TYPE_REGEXP`。

- **用 StrCat 而非 StringPrintf 来格式化 IP 地址。**（ipaddress.cc）原 `IPAddress::ToString` 对 IPv4 也用 `inet_ntop`，且 `IPAddressToURIString`、`SocketAddress::ToString` 用 `StringPrintf("[%s]", …)`、`StringPrintf(":%u", port_)`。新实现对 `AF_INET` 手工取出 4 个字节 `a1..a4` 并 `StrCat(a1, ".", a2, ".", a3, ".", a4)`；URI 包裹改为 `StrCat("[", ip.ToString(), "]")`；套接字地址改为 `StrCat(IPAddressToURIString(host_), ":", port_)`。

> **Rust 备注**：
> - 热路径上避免 `format!`（会分配新 `String`），改用 `write!(&mut reused_string, …)` 写入可复用缓冲区，或直接用整数 `to_string`/手工拼接。
> - 前缀匹配用 `str::starts_with` / `slice::starts_with` 取代正则。
> - 编译期常量分派可用泛型 + `const` 泛型参数或 `#[inline]` 让常量传播生效，等价于 `VLOG` 的按级别特化。

### 用缓存避免重复工作

- **基于大型序列化 proto 的预计算指纹进行缓存。**（dp_ops.cc）原代码每次都 `ParseFromStringPiece` 解析 `InputOutputMappingProto` 再 `ParseMapping`。新实现先取预计算指纹 `mapping_proto_fp = GetAttrMappingProtoFp(state)`，在 `absl::MutexLock` 保护下查一个全局 `absl::flat_hash_map<uint64, unique_ptr<ProgramIOMetadata>> fp_to_iometa`：命中则直接复用 `io_metadata_`；未命中才解析（`DCHECK_EQ(mapping_proto_fp, Fingerprint(serial_proto))`）、`ParseMapping` 并把结果存入缓存。

> **Rust 备注**：缓存可用 `HashMap<u64, Arc<Meta>>` 配合 `Mutex`/`RwLock`，或用 memoize 类库；键采用预计算指纹（如 `u64`）避免对大对象重复哈希/解析。

### 让编译器的工作更轻松

由于编译器必须对代码整体行为做保守假设，或者可能没有在速度与体积之间做出正确权衡，它在穿透多层抽象进行优化时可能力不从心。应用程序员往往更了解系统行为，可以通过把代码改写到更低层次来帮助编译器。不过，只有在性能剖析确实显示出问题时才这样做，因为编译器通常自己就能做对。查看性能关键例程生成的汇编代码，有助于判断编译器是否 " 做对了 "。Pprof 提供了非常有用的源码与反汇编交错展示，并标注了性能数据。

一些可能有用的技巧：

1. 在热函数中避免函数调用（让编译器省去栈帧建立的开销）。
2. 把慢路径代码移入一个单独的、以尾调用方式调用的函数。
3. 在密集使用前，把少量数据复制到局部变量中。这可以让编译器假定它与其他数据不存在别名，从而可能改善自动向量化与寄存器分配。
4. 对非常热的循环进行手工展开（hand-unroll）。

- **用指向底层数组的裸指针替换 absl::Span，从而加速 ShapeUtil::ForEachState。**（shape_util.h）原 `ForEachState` 的 `base`/`count`/`incr` 三个成员是 `const absl::Span<const int64_t>`，且析构函数非内联。新实现把它们改为裸指针 `const int64_t* const base/count/incr`（指向传入 span 的底层数组），并把析构函数改为 `inline ~ForEachState() = default;`。

- **手工展开循环冗余校验（CRC）计算循环。**（crc.cc）原实现 " 每次处理 4 字节 "（一个 `while` 循环内做一次查表步骤）。新实现用宏 `STEP { uint32 c = l ^ WORD(p); p += 4; l = table3_[c&0xff] ^ table2_[(c>>8)&0xff] ^ table1_[(c>>16)&0xff] ^ table0_[c>>24]; }`，先 " 每次处理 16 字节 "——在 `while ((e-p) >= 16)` 中连续调用 4 次 `STEP`；随后再用 `while ((p+4) <= e) STEP;` 处理 4 字节块；最后逐字节处理剩余部分。

- **解析 Spanner 键时每次处理四个字符。**（key.cc）三点改动：(1) 手工展开循环，每次处理四个字符而非用 `memchr`；(2) 手工展开用于寻找名字各分隔段的循环；(3) 对以 `#` 分隔的名字，从后向前查找各分隔段（而非从前向后），因为名字的第一部分很可能最长。原 `InitSeps` 用 `for` 循环三次 `memchr(s, '#', …)`。新实现引入 `ScanBackwardsForSep`（在 `while (p >= base+4)` 中一次检查 `p[0]`、`p[-1]`、`p[-2]`、`p[-3]` 是否为 `#`，否则 `p -= 4`；收尾逐字节回退），再从 `limit-1` 向前三次调用它填 `seps_[2]`、`seps_[1]`、`seps_[0]`。注释说明：目录名可能很长且肯定不含 `#`，所以从末尾往前扫更划算。

- **通过把 ABSL_LOG(FATAL) 改为 ABSL_DCHECK(false) 来避免栈帧建立开销。**（arena_cleanup.h）在 `ABSL_ATTRIBUTE_ALWAYS_INLINE` 的 `Size(Tag tag)` 中，`switch` 的 `default` 分支原为 `ABSL_LOG(FATAL) << "Corrupted cleanup tag: " << …`，新实现改为 `ABSL_DCHECK(false) << …`（返回 `sizeof(DynamicNode)`），从而避免为该 always-inline 函数引入日志导致的栈帧建立成本。

> **Rust 备注**：
> - 把慢路径拆成 `#[cold] #[inline(never)]` 函数，等价于 " 移入单独的尾调用函数 "。
> - 在热用前把小块数据 `let base = *slice_ref;` 复制到局部变量，可减少别名假设、利于自动向量化与寄存器分配。
> - Rust 同样可以手工展开循环；`chunks_exact(N)` 是惯用写法（例如 `for chunk in bytes.chunks_exact(16) { … }`，再用 `.remainder()` 处理尾部），保留原文 " 手工展开 " 的意图。**注意：不要声称 CRC 的查表运算会自动向量化为 SIMD**——查表本质上难以向量化，这里的收益来自减少循环开销与分支，而非 SIMD。
> - 把不可达的致命分支写成 `debug_assert!(false, …)` 配合 `unreachable!()`，可在 release 构建中避免昂贵的 panic/frame 建立路径。

### 降低统计收集的成本

要在统计等系统行为信息的价值与维护这些信息的成本之间做权衡。额外信息常常能帮助人们理解并改进系统的高层行为，但维护起来也可能代价高昂。

无用的统计可以彻底删除。

- **停止在 SelectServer 中维护关于 alarm 与 closure 数量的昂贵统计。**（selectserver.h / selectserver.cc）这是把设置一个 alarm 的耗时从 771 ns 降到 271 ns 的一系列改动的一部分。删除了 `num_alarms_stat_`、`num_closures_stat_`（`MinuteTenMinuteHourStat`）成员；`AddAlarmInternal` 中把 `alarms_->insert(alarm); num_alarms_stat_->IncBy(1);` 改为 `alarms_->Add(alarm);`；`RemoveAlarm` 中把 `alarms_->erase(alarm); num_alarms_stat_->IncBy(-1);` 改为 `alarms_->Remove(alarm);`。

统计或其他属性常常可以只针对系统处理的元素样本来维护（例如 RPC 请求、输入记录、用户）。许多子系统采用这种做法（tcmalloc 分配跟踪、/requestz 状态页、Dapper 采样）。采样时，可在合适时降低采样率。

- **只为一部分抽样的 doc info 请求维护统计。**（generic-leaf-stats.cc）采样使我们能在绝大多数请求上避免触碰 39 个直方图与 MinuteTenMinuteHour 统计。原代码在每个请求上都去更新 39 个直方图；新代码用 `if (TryLockToUpdateHistogramsDocInfo(docinfo_stats, bucket)) { …更新 39 个直方图…; bucket->lock.Unlock(); }` ——只有当应当为本次请求采样时它才返回 true 并抓住 `bucket->lock`。

- **降低采样率并加快采样判定。**（packet_executor.cc）此改动把采样率从 1/10 降到 1/32；此外只为被采样的事件保留执行时间统计，并通过使用 2 的幂取模来加快采样判定。这段代码在 Google Meet 视频会议系统中对每个数据包都会执行，在 COVID 疫情初期用户迅速转向线上会议、容量需求激增时，需要做性能优化才能跟上。原 `ScopedPerformanceMeasurement` 构造函数中 `if (packet_executor->closures_executed_ % 10 == 0)` 才取 `ThreadCPUUsage`（该调用超过 400 ns，约为 `absl::Now` 的 30 倍），新实现改为 `% 32 == 0`；析构函数中把 `closure_execution_time->Record(…)` 移入 `if (thread_cpu_usage_start_.has_value())` 内，仅对采样事件记录。基准结果：`BM_PacketOverhead_mean` 由 224 ns 降到 85 ns，改善 **+62.0%**（Run on 40 X 2793 MHz CPUs; Intel Ivybridge, 2020-03-24）。

> **Rust 备注**：无用统计直接删除；采样计数用 `count & (N-1) == 0`（N 为 2 的幂）取代取模，等价于原文的 "power-of-two modulus" 快速判定。热路径上的原子统计计数尽量避免，或改为按需/采样聚合。

### 避免在热代码路径上打日志

即使某条日志语句的日志级别并不会真正输出任何内容，日志语句也可能代价高昂。例如 `ABSL_VLOG` 的实现至少需要一次加载和一次比较，这在热代码路径上可能成为问题。此外，日志代码的存在还可能抑制编译器优化。可以考虑从热代码路径上彻底移除日志。

- **从内存分配器的核心逻辑中移除日志。**（gpu_bfc_allocator.cc）这是一个更大改动的一小部分。从 `GPUBFCAllocator::SplitChunk` 中删除 `VLOG(6) << "Adding to chunk map: " << new_chunk->ptr;`，从 `DeallocateRawInternal` 中删除 `VLOG(6) << "Chunk at " << c->ptr << " no longer in use";`。

- **在嵌套循环外预计算日志是否启用。**（image_similarity.cc）原双重循环内部反复调用 `if (VLOG_IS_ON(3)) { … }`。新实现在循环外先算 `const bool vlog_3 = DEBUG_MODE ? VLOG_IS_ON(3) : false;`，循环内改用 `if (vlog_3)`。基准结果（Run on 40 X 2801 MHz CPUs; Intel Ivybridge, 2016-05-16）：
  - BM_NCCPerformance/16：29104 → 26372，**+9.4%**
  - BM_NCCPerformance/64：473235 → 425281，**+10.1%**
  - BM_NCCPerformance/512：30246238 → 27622009，**+8.7%**
  - BM_NCCPerformance/1k：125651445 → 113361991，**+9.8%**
  - BM_NCCLimitedBoundsPerformance/16：8314 → 7498，**+9.8%**
  - BM_NCCLimitedBoundsPerformance/64：143508 → 132202，**+7.9%**
  - BM_NCCLimitedBoundsPerformance/512：9335684 → 8477567，**+9.2%**
  - BM_NCCLimitedBoundsPerformance/1k：37223897 → 34201739，**+8.1%**

- **预计算日志是否启用，并在辅助例程中复用该结果。**（periodic_call.cc）原 `ScheduleNextAlarm`/`ScheduleAlarm` 等函数各自内联多条 `VLOG(1) << …`。新实现在顶层先算一次 `const bool vlog_1 = VLOG_IS_ON(1);`，把 `vlog_1` 作为参数逐层传入 `ScheduleNextAlarm(…, bool vlog_1)`、`ScheduleAlarm(…, bool vlog_1)`，每处日志都包在 `if (vlog_1) { VLOG(1) << …; }` 中，从而只判定一次。

> **Rust 备注**：使用 `log` / `tracing` 时，用 `log_enabled!(Level::Debug)` / `tracing::enabled!(…)` 在嵌套循环外预计算一次并复用；并通过编译期最大日志级别（如 `log` 的 `max_level_*` feature 或 `tracing` 的 `max_level_*`）在 release 构建中彻底剔除热路径上的日志代码，避免运行期加载与比较开销。

## 代码体积考量 (Code size considerations)

性能不仅仅是运行速度。有时值得考虑软件选择对生成代码体积的影响。代码体积过大意味着更长的编译与链接时间、臃肿的二进制文件、更多内存占用、更大的指令缓存压力，以及对分支预测器等微架构结构的负面影响。在编写会被大量复用的底层库代码，或编写预期会针对许多类型实例化的模板代码时，尤其需要留意。

降低代码体积的技巧因语言而异。以下是一些对 C++（容易过度使用模板与内联）行之有效的做法。

### 精简被广泛内联的代码

被广泛调用的函数一旦与内联结合，会对代码体积产生显著影响。

- **加速 TF_CHECK_OK** — 避免构造 Ok 对象，并把致命错误信息的复杂格式化移到 out-of-line，而非在每个调用点内联，从而节省代码空间。
- **将每个 RETURN_IF_ERROR 调用点缩减 79 字节** — 为 RETURN_IF_ERROR 专门加适配器类，在快路径上不构造/析构 StatusBuilder，不再内联相关方法，并避免多余的 `~Status` 调用。
- **让 CHECK_GE 提速 4.5X 并把代码从 125 字节缩到 77 字节** — 去掉 CheckOpString 的析构函数（反正非空时马上要 LOG(FATAL)），并为 int/int 场景提供 out-of-line 的字符串构造辅助函数。

> Rust 备注：宏与 `#[inline]` 的膨胀效应同样存在。把错误格式化路径抽成独立的非内联函数（`#[cold]` / `#[inline(never)]`），只让快路径保持内联。

### 谨慎内联

内联常能提升性能，但有时只会增大代码体积却没有相应回报，甚至因指令缓存压力上升而变慢。

- **减少 TensorFlow 中的内联** — 停止内联许多非性能敏感函数（如错误路径、op 注册代码），并把部分性能敏感函数的慢路径移出内联。使典型二进制中 tensorflow 符号体积减少 12.2%（8814545 → 7740233 字节）。
- **protobuf 消息长度编码 out-of-line** — 对 ≥ 128 字节的消息，不再内联昂贵的长度编码代码，而是调用共享的 out-of-line 例程；既让重要的大型二进制更小，也更快。
- **缩减 absl::flat_hash_set/map 代码体积** — 把不依赖具体哈希表类型的代码抽成公共的非内联函数，审慎放置 NOINLINE 指令，并将部分慢路径移出内联，使一些大型二进制缩小约 0.5%。

### 减少模板实例化

模板代码会针对模板参数的每种组合被重复实例化。

- **用普通参数替换模板参数** — 把一个以 `bool` 为模板参数的大型例程改为把该 bool 作为普通参数传入（它只用于二选一地选字符串常量，运行时判断即可），使该例程的实例化数从 287 降到 143。
- **把臃肿代码从模板构造函数移到非模板共享基类构造函数** — 同时把实例化从每种 `<T, Device, Rank>` 组合各一份，减少为每个 `<T>` 和每个 `<Rank>` 各一份。

> Rust 备注：泛型单态化会为每组类型参数生成一份代码。把与类型无关的臃肿逻辑抽成非泛型内核函数，让泛型外壳只做薄薄的转发。

### 减少容器操作

留意 map 等容器操作的影响——每次这类调用都可能产生大量生成代码。

- **把一连串 map 插入合并为单次批量插入** — 初始化 emoji 字符哈希表时，将逐条插入改为一次 bulk insert，使被众多二进制链接的库中相关文本从 188KB 降到 360 字节。
- **停止内联 InlinedVector 的重度使用者** — 把一个被内联的超长例程从 .h 移到 .cc（此处内联无实际性能收益）。

## 并行化与同步 (Parallelization and synchronization)

### 利用并行

现代机器核心众多，却常被闲置。因此昂贵的工作可以通过并行来更快完成。最常见的做法是并行处理不同项并在完成后合并结果；通常先把项分批，以摊薄逐项并行的开销。

- **四路并行使 token 编码速率提升约 3.6x** — 用线程池并行编码各区域的 token。
- **并行化使解码性能提升 5x** — 把逐 cluster 的串行解码改为向 executor 提交每个 cluster 的子任务，再统一等待通知并收集结果。

系统层面的效果应仔细度量——若没有空闲 CPU，或内存带宽已饱和，并行化可能无助甚至有害。

> Rust 备注：CPU 密集的批处理优先用 `rayon` 的 `par_iter()`；它自带工作窃取与结果合并，改造成本低。

### 摊薄锁获取

避免细粒度加锁以降低热路径上 Mutex 操作的开销。注意：仅当此改动不会加剧锁竞争时才可采用。

- **一次加锁释放整棵查询节点树** — 不再为树中每个节点反复获取锁，而是获取一次锁后递归调用 `ReleaseLocked`（内部断言持锁）释放整棵子树。

> Rust 备注：对应做法是在递归入口获取一次锁，内部走 " 已持锁 " 的私有方法，避免对子树每个节点重复 lock。

### 保持临界区简短

避免在临界区内做昂贵工作。尤其警惕那些看似无害、实则可能发起 RPC 或访问文件的代码。

- **减少临界区触及的缓存行数** — 通过精心的数据结构调整，把每节点类型属性预计算为 NodeItem 内的位标志，并让 ActivateNodes 使用目标节点的 NodeItem 而非触及 `*item->node` 字段；通常从 `~2 + O(出边数)` 条缓存行降到 1~2 条，使某 ML 训练运行提速 3.3%。
- **不要持锁时发 RPC** — 在锁内只读出所需状态（是否记录进度、步数）到局部变量，出锁后再执行 `StartRecordProgress`。

此外要警惕在 Mutex 解锁前运行的昂贵析构函数（常由 `~MutexUnlock` 触发）；把带昂贵析构的对象声明在 MutexLock 之前可能有帮助（前提是线程安全）。

> Rust 备注：在锁内 copy-out 所需数据，然后在做 I/O 前显式 `drop(guard)`；且切勿跨 `.await` 持有 `std::sync::Mutex`。

### 通过分片减少竞争

有时一个受 Mutex 保护、竞争激烈的数据结构可以安全地拆成多个分片，每个分片各持一把 Mutex（前提是分片间没有跨分片不变式）。

- **把缓存分成 16 片** — 按 key 的哈希高位选择分片，使多线程负载下的吞吐提升约 2x。
- **给 Spanner 的调用跟踪结构分片** — 通过 `LockedShard(tid)` 接口以线程安全方式访问某事务对应的 ActiveCallMap。
- **修正分片选择所用的信息** — 若用哈希的某些位做分片选择，而这些位随后又被底层哈希表复用，会导致分布倾斜、性能变差；改为把哈希与 42 组合（`hasher({index, 42})`）后再取模，避免与底层哈希表用相同的位。
- **把 Spanner 调用跟踪结构分成 64 片** — 每片独立 mutex，事务映射到唯一分片；在 8192 fibers 的基准下整体墙钟时间减少 69%。

若相关数据结构是 map，可考虑改用并发哈希表实现。

> Rust 备注：per-key 并发 map 用 `dashmap`；或自建 `Vec<Mutex<Shard>>` 并按 key 哈希选片。

### SIMD 指令

探索用现代 CPU 上的 [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data) 指令一次处理多个数据项能否带来提速（例如后文 Bulk Operations 一节中关于 `absl::flat_hash_map` 的讨论）。

> Rust 备注：可用 `std::arch` 内建函数（需 `unsafe` 与目标特性）、可移植的 `std::simd`（nightly），或第三方 `wide` crate。

### 减少伪共享

若不同线程访问不同的可变数据，考虑把这些数据项放到不同缓存行，例如 C++ 中用 `alignas` 指令。但这类指令容易误用，还可能显著增大对象体积，务必用性能度量来证明其必要性。

- **将频繁改写的字段隔离到独立缓存行** — 在 histogram 中对 `buckets_`、`min_/max_/count_/sum_/sum_of_squares_` 等频繁改写字段加 `alignas(ABSL_CACHELINE_SIZE)`，避免与其他字段发生伪共享。

> Rust 备注：用 `#[repr(align(64))]` 对齐结构体，或用 `crossbeam_utils::CachePadded<T>` 包裹热点字段。

### 减少上下文切换频率

- **小工作项就地处理，不下发到设备线程池** — 在 cast_op 中，当 `o.size() * (sizeof(Tin) + sizeof(Tout)) < 16384` 时直接内联做小型 cast，否则才提交到设备执行，避免为小任务付出线程池调度与上下文切换的代价。

### 使用带缓冲的通道做流水线

通道可以是无缓冲的，这意味着写入方会阻塞直到读取方准备好取走一项。无缓冲通道在用于同步时很有用，但在用于提升并行度时并不合适——若要靠通道构建流水线以增加并行，应使用带缓冲的通道。

> Rust 备注：带缓冲通道用 `std::sync::mpsc::sync_channel(n)` 或 `crossbeam-channel` 的 bounded(n)；容量 n 决定写入方在阻塞前可积压多少项。

### 考虑无锁方法

有时无锁数据结构相较于传统的互斥锁保护结构能带来差异。但直接操作原子变量可能很危险，应优先选用更高层的抽象。

- **用无锁 map 管理 RPC channel 缓存** — RPC stub 缓存中的条目每秒被读取数千次而极少修改，切换到合适的无锁 map 使查找延迟降低 3%-5%。
- **用固定词典 + 无锁哈希表加速 IsValidTokenId 判定** — 把受 mutex 保护的 `dense_hash_map` 换成 `LockFreeHashMap`，读访问走 epoch GC 的 EnterFast/LeaveFast，写方定期 GC 已删除条目。

> Rust 备注：per-key 的读多写少 map 用 `dashmap`；`arc-swap` 只适合对整个值做读多写少的原子替换，并不适合作为 per-key map。

## Protocol Buffer 建议 (Protocol Buffer advice)

Protobuf 是数据的便捷表示形式，尤其适合通过网络传输或持久化存储的数据，但可能带来显著的性能代价。例如：一段填充 1000 个点、再对 Y 坐标求和的代码，从 protobuf 改为 C++ `std::vector` of structs 后，速度提升达 **20 倍**（基准 `BenchmarkIteration` 从 17.4µs 降到 0.8µs，-95.30%）。此外 protobuf 版本还会给二进制增加几 KB 的代码与数据，累积起来造成 i-cache/d-cache 压力。

- **不必要时不要用 protobuf**：鉴于上面 20 倍的差距，如果某份数据从不序列化或解析，就不应放进 protobuf。它的目的在于方便序列化/反序列化；若你只想要 `DebugString`、可拷贝性这类便利，别为此付出代码体积、内存和 CPU 的开销。Rust 对应：直接用 `Vec<struct>`，而非 prost 生成的类型。
- **避免不必要的 message 层级（尽量扁平化）**：额外的消息层级会带来内存分配、函数调用、cache miss、更大的序列化消息等开销。与其 `Foo { Bar bar = 1 } / Bar { Baz baz = 1 } / Baz { int32 count = 1 }`，不如直接 `Foo { int32 count = 1 }`。扁平形式无需遍历消息树，所有 protobuf 操作（解析、序列化、求 size 等）都更便宜。
- **频繁出现的字段用小 field number**：field number 与 wire type 组合采用变长整数编码——field number 在 **1–15 之间占 1 字节**，**16–2047 之间占 2 字节**（≥2048 通常应避免）。可为性能敏感的 protobuf 预留一些小 field number 供将来扩展。
- **谨慎选择整数类型**：一般用 `int32`/`int64`；对哈希码等大值用 `fixed32`/`fixed64`；对经常为负的值用 `sint32`/`sint64`。varint 对小整数省空间但解码更贵，对负数或大值反而更占空间——此时 `fixed32/64`（而非 `uint32/64`）体积更小且编解码便宜得多。
- **proto2 用 `[packed=true]` 打包重复数值字段**：proto2 默认把 repeated 值序列化为一串 (tag, value) 对，每个元素都要解 tag，效率低。packed 形式先写 payload 长度再写无 tag 的值。对定宽类型（`fixed32/64`、`float`、`double` 等）效果最好，可预知总长度、避免重分配；变长 varint 仍可能付出重分配代价。proto3 中 repeated 字段默认已 packed。
- **二进制/大值用 `bytes` 而非 `string`**：`string` 存 UTF-8 文本，有时需要校验；`bytes` 可存任意字节序列（非文本数据），既更合适也更高效（避免 UTF-8 校验）。
- **考虑 `string_type = VIEW` 以避免拷贝**：解析时拷贝大 string/bytes 字段代价很高，可用该标注避免。例如 `bytes jpeg_encoding = 4 [features.(pb.cpp).string_type=VIEW];`。像 `ParseFromStringWithAliasing` 这类接口用视图引用原始 backing string，而不拷贝大块二进制——但 backing string（即序列化缓冲区）必须比含别名的 protobuf 实例活得更久。Rust 对应：借用 `&[u8]`/`&str`（非拥有，生命周期绑定到 buffer）。
- **大字段考虑 `[ctype=CORD]` 降低拷贝成本**：把大 `bytes`/`string` 字段标为 `[ctype=CORD]`，表示从 `std::string` 变为 `absl::Cord`——引用计数 + 树形存储，减少拷贝与追加成本；若 protobuf 序列化到 cord，解析这类字段可免拷贝。性能取决于长度分布与访问模式，需用基准验证。Rust 对应：`bytes::Bytes`（引用计数共享）。注意 VIEW 与 Cord 不可混为一谈。
- **考虑在内存中也以序列化形式存储 protobuf**：内存中的 protobuf 对象内存占用很大（常达 wire 格式的 5 倍），且可能分散在多条缓存行。若应用要长期保活大量 protobuf，考虑存序列化形式。
- **避免 protobuf map 字段**：其性能问题通常超过它带来的少量语法便利。优先用从 protobuf 内容初始化的非 protobuf map，例如把 `map<string, bytes> env_variables = 5;` 改为 `message Var { string key = 1; bytes value = 2; }` + `repeated Var env_variables = 5;`。
- **用只含字段子集的消息定义**：若只需访问大消息的少数字段，可定义模仿原类型、但只声明你关心字段的消息（如 100 字段的 `FullMessage` 对应只含 field3、field88 的 `SubsetMessage`）。把序列化的 `FullMessage` 解析成 `SubsetMessage`，只解析这两个字段，其余当作未知字段；必要时用丢弃未知字段的 API 进一步提速。
- **在 C++ 中使用 protobuf arena**：可节省分配/释放成本，尤其对含 repeated、string 或 message 字段的 protobuf。message/string 字段是堆分配的（即便顶层对象在栈上）；arena 摊薄分配成本、使释放几乎免费，并通过连续内存块提升局部性。Rust 对应：复用消息对象来替代 arena。
- **保持 .proto 文件小**：不要在单个 .proto 里塞太多消息。一旦依赖其中任何东西，整个文件都会被链接器拉入（即便大多未用），增加构建时间与二进制体积。可用 extensions 和 `Any` 避免对含大量消息类型的大 .proto 产生硬依赖。
- **尽量复用 protobuf 对象**：把 protobuf 对象声明在循环外，使其已分配的存储能跨迭代复用。

## Rust 特定建议（对应原文 C++-Specific advice）

### absl::flat_hash_map（及 set）→ std `HashMap`
Abseil 哈希表通常胜过 `std::map`、`std::unordered_map`。示例：`LanguageFromCode` 从 `__gnu_cxx::hash_map` 换成 `absl::flat_hash_map`，基准 `BM_CodeToLanguage` 从 19.4ns 降到 10.2ns（-47.47%）；stats publish/unpublish 与 SelectServer alarm 表也从 `hash_map` 换成 `dense_hash_map`（今天会用 `flat_hash_map`）。Rust 对应：标准库 `HashMap`（基于 hashbrown，采用 SIMD 探测的 Swiss Table）。

### absl::btree_map / absl::btree_set → `BTreeMap` / `BTreeSet`
每个树节点存多个条目：减少指向子节点的指针开销，且同节点的键值连续存放、cache 效率更佳。示例：把一个高频使用的 work-queue 从 `std::set<WorklistItem>` 换成 `absl::btree_set<WorklistItem>`。

### util::bitmap::InlinedBitVector → `bitvec` / `fixedbitset`
可把短位向量内联存储，常优于 `std::vector<bool>` 等位图类型。示例：把 `vector<bool> live_reads(nreads)` 换成 `util::bitmap::InlinedBitVector<4096>`，并用 `FindNextSetBit` 直接跳到下一个置位项，避免逐位遍历。

### absl::InlinedVector → `SmallVec` / `ArrayVec`
把少量元素内联存储（数量由第二个模板参数配置），小向量可获得更好 cache 效率并完全避免分配 backing 数组。示例：把 `std::vector<InstructionRecord> instructions_` 换成 `absl::InlinedVector<InstructionRecord, 2>`。

### gtl::vector32 → Rust 无直接对应
使用只支持 32 位大小的定制 vector 类型来省空间。示例：`std::vector<FamilyId>` 换成 `gtl::vector32<FamilyId>`（读取接口改用 `absl::Span`），该简单类型改动在 Spanner 省下约 **8TiB** 内存。Rust 无直接对应：用 `u32` 索引替代。

### gtl::small_map → `Vec<(K, V)>` 线性查找
用内联数组存放至多若干个唯一键值对，超容量后自动升级为用户指定的 map 类型。示例：`gtl::flat_hash_map<int, TFLiteContext*>` 改为 `gtl::small_map<gtl::flat_hash_map<int, TFLiteContext*>>`。Rust 对应：小规模用 `Vec<(K, V)>` 线性查找。

### gtl::small_ordered_set → 有序 `Vec` + 二分查找
关联容器（如 `std::set`、`absl::btree_multiset`）的优化：先用定长数组存一定数量元素，超容量后退回 set/multiset。对通常很小的集合，比直接用为大数据集优化的 `set` 快得多，缩小 cache 足迹并缩短临界区。示例：`std::set<ParsedRtpTransport*>` 换成 `gtl::small_ordered_set<std::set<ParsedRtpTransport*>, 10>`。Rust 对应：有序 `Vec` + 二分查找。

### gtl::intrusive_list → `intrusive-collections` crate 或 `Vec`+ 索引
链接指针内嵌在类型 T 的元素中的双向链表；相比 `std::list<T*>`，每个元素省一条缓存行 + 一次间接。示例：把 `std::set<int64> inflight_requests_` 换成继承 `gtl::intrusive_link<SeqNum>` 的 `SeqNum` 元素 + `gtl::intrusive_list<SeqNum>`。Rust 对应：`intrusive-collections` crate，或 `Vec` + 索引。

### 限制 absl::Status / absl::StatusOr 使用 → `Result` / `Option`
即便 `absl::Status`/`StatusOr` 相当高效，成功路径上仍有非零开销，故热点函数若无需返回有意义的错误细节（甚至永不失败）应避免使用。保留四个示例：
- `RoundUpToAlignment()` 去掉 `absl::StatusOr<int64>` 返回类型，改为返回裸 `int64`，把 `TPU_RET_CHECK` 换成 `DCHECK` 前置条件检查。
- 新增 `ShapeUtil::ForEachIndexNoStatus`，visitor 从 `StatusOr<bool>` 改为返回 `bool`，避免为张量每个元素创建 Status 返回对象。
- `TF_CHECK_OK` 改为通过 `TfCheckOpHelper`/`TfCheckOpHelperOutOfLine` 检测 `v.ok()`，避免为测试 ok() 而构造 OK 对象。
- 从 RPC 热路径移除 `StatusOr`（`GetRawPrivacyContext` 改为返回 `enum class Result` + 出参），修复了此前改动引入的 **14% CPU regression**。

Rust 备注：`FxHashMap`/`ahash` 等更快的 hasher 可显著加速，但它们不抗 HashDoS——仅在输入可信时使用。

## 批量操作 (Bulk operations)

尽量一次处理多个条目，而非逐个处理。

- **flat_hash_map 用单条 SIMD 指令**在一组 slot 中比较每个键的一个哈希字节（`Match` 用 `_mm_cmpeq_epi8` + `_mm_movemask_epi8` 一次比对整组控制字节）。Rust 对应：std/hashbrown 已内建此机制（Swiss Table）。
- **用单次操作处理许多字节再做修正**，而非逐字节判断该做什么。示例：`ordered-code.cc` 里逐字节移位的循环被替换为 `BigEndian::Store` 一次写入 + `OrderedNumLength` 计算长度。Rust 对应：`chunks_exact` / `bytemuck` / `std::simd`。
- **一次解码一组整数**（例如 4 个）比逐个更快。示例：GroupVarInt 格式一次编解码 4 个变长整数、占 5–17 字节；解码一组 4 个的耗时约为逐个 varint 解码 4 个的 **1/3**。相关地，`KBitStreamEncoder`/`KBitStreamDecoder` 一次编解码 4 个 k-bit 数——因 K 编译期已知，可假定流总是字节对齐（偶 k）或半字节对齐（奇 k），非常高效。

## 综合运用多种技巧的 CL (CLs that demonstrate multiple techniques)

有时一个 CL 会包含多项性能改进，同时用到前文介绍的许多技巧。在某个部分被确认为瓶颈之后，研读这类 CL 中的改动方式，往往是培养 " 如何系统性地为系统某部分提速 " 这种思维方式的好途径。

- **将 GPU 内存分配器提速约 40%(Speed up GPU memory allocator by ~40%)** — 对 GPUBFCAllocator 的分配/释放速度带来 36–48% 的提速。综合运用了多项技巧：用句柄编号 (ChunkHandle，4 字节) 代替 `Chunk*` 指针 (8 字节) 并把 Chunk 存进 `vector<Chunk>`；为空闲 Chunk 维护自由链表以避免堆分配/释放并使内存连续；用按 log₂(byte_size/256) 索引的数组代替 `std::set`+`lower_bound` 来定位 bin(几次位运算即可，且 Bin 数据结构连续存放、减少跨核缓存行迁移)；为 AllocateRaw 增加快路径以绕开 retry_helper_ 与 `std::function` 分配；注释掉大部分 VLOG。基准显示 BM_Allocation 从 347ns 降到 184ns(+47.0%)，多线程场景亦有约 20–48% 提升；端到端使 ptb_word_lm 从 8036 词/秒提升到 8272 词/秒 (+2.9%)。
  - Rust 备注：句柄/索引取代指针 = 用 `Vec<Chunk>` + `u32` 索引 (避免 `Box`/引用与所有权纠缠)；自由链表 = arena/对象池模式 (参考 `slab`、`bumpalo`)；log₂ 分桶 = `usize::leading_zeros`/`ilog2` 位运算；快路径避免 `std::function` 分配对应 Rust 中避免装箱闭包 `Box<dyn Fn>`。

- **通过一组杂项改动将 Pathways 吞吐提升约 20%(Speed up Pathways throughput by ~20%)** — 将多个专用的快速描述符解析函数统一为单一的 ParsedDescriptor 类以避免昂贵的完整解析；把若干 protobuf 字段从 string 改为 bytes(省去 UTF-8 校验与错误处理);DescriptorProto.inlined_contents 由 Cord 改为 string; 若干处以 flat_hash_map 替换 `std::unordered_map`; 新增 MemoryManager::LookupMany 供 Stack op 批量查找以减少加锁等准备开销; 移除 TransferDispatchOp 中不必要的字符串创建。在同一进程内传输 1000 个 1KB 张量的批次：从 227.01 步/秒提升到 272.52 步/秒 (+20% 吞吐)。
  - Rust 备注：flat_hash_map 对应 `hashbrown`(即标准库 `HashMap` 底层)；批量接口 `LookupMany` 对应 Rust 中一次性传入切片、分摊加锁/边界成本的 API 设计。

- **通过一系列改动使 XLA 编译器性能提升约 15%(~15% XLA compiler performance improvement)** — 在 SortComputationsByContent 中当 `a==b` 时直接返回 false，避免对长计算字符串做序列化与指纹计算; 将 CHECK 改为 DCHECK 以少碰一条缓存行; 避免在 CoreSequencer::IsVectorSyncHoldSatisfied() 中昂贵地复制首条指令; 重写 HloComputation::ToString/ToCord 使主要工作以追加到 `std::string` 而非 Cord; 把 PerformanceCounterSet::Increment 从两次哈希查找减为一次; 精简 Scoreboard::Update。对某个重要模型的 XLA 编译时间总体提速 14%。
  - Rust 备注：DCHECK 对应 `debug_assert!`；追加到字符串 = 复用 `String::push_str` 缓冲；单次哈希查找 = 用 `entry` API 避免二次探测。

- **加速 Google Meet 应用代码中的底层日志 (Speed up low level logging in Google Meet)** — 优化位于每个数据包关键路径上的 ScopedLogId：移除仅用于检查不变式的 `LOG_EVERY_N(ERROR, …)` 消息; 去掉这些语句后 PushLogId/PopLogId 已足够小从而内联; 用固定大小为 4 的数组加一个 `int size` 变量替代 `InlinedVector<…>`(因为本就从不超过 4，InlinedVector 的通用性属于过度设计)。基准显示 BM_ScopedLogId 从 8ns 降到 4ns(各线程数下约 +44%~+53%)。
  - Rust 备注：`InlinedVector` 对应 `smallvec`；此处更进一步用固定 `[T; 4]` + `usize` 长度替代小向量。

- **通过改进 Shape 处理将 XLA 编译时间减少约 31%(Reduce XLA compilation time by ~31%)** — 多项改动：优化 ShapeUtil::ForEachIndex 迭代 (在 ForEachState 中只存数组指针而非完整 span、预先构造 indexes_span、保存 indexes_ptr 与 minor_to_major 指针以用简单数组操作替代 `vector::operator[]`、内联构造函数与 IncrementDim); 为无需返回 Status 的调用点引入 ForEachIndexNoStatus 变体 (回调返回裸 bool，避免每元素一次昂贵的 `StatusOr<bool>` 析构); 多方面优化 LiteralBase::Broadcast(引入按基本类型字节大小特化的模板 BroadcastHelper 让 memcpy 可被良好优化、把每次 Broadcast 约 5+num_dimensions+num_result_elements 次虚调用减到仅一次 shape() 调用、特判源维度为 1、用裸指针替代 vector 下标、引入传入 minor_to_major 的三参数 MultiDimensionalIndexToLinearIndex); 在 ShardingPropagation::GetShardingFromUser 中对 kTuple 情形延迟调用 GetSubSharding，使某次冗长编译在此函数中的 CPU 时间从 43.7s 降到 2.0s; 并新增基准。基准显示 BM_ForEachIndex/2 提升 +16.8%;Broadcast 性能提升约 58%(BM_BroadcastVectorToMatrix 各规模 +57.3%~+59.0%); 对某大语言模型做 AOT 编译整体从 573 秒降到 465 秒 (+19%)，两个最大 XLA 程序的编译时间从 141s+143s=284s 降到 99s+95s=194s(+31%)。
  - Rust 备注：`StatusOr<bool>` 每元素析构开销 = Rust 中避免在热循环里返回带析构器的 `Result<_, E>`(尤其 `E` 非 `Copy` 时)；按字节大小特化模板 = 泛型单态化 + `const N: usize`；裸指针替代下标 = 用切片迭代器或 `get_unchecked`(需 `unsafe`) 消除边界检查。

- **在 Plaque(分布式执行框架) 中将大型程序的编译时间减少约 22%(Reduce compilation time by ~22% in Plaque)** — 小幅调整实现约 22% 提速：加速检测两节点是否共享公共源 (先把一个节点的源放入哈希表，再遍历另一节点的源做查表，取代此前 " 取有序源 + 有序求交 "); 复用同一张临时哈希表; 生成编译后 proto 时用以 `pair<package, opname>` 为键的单层 btree 取代 btree 套 btree; 在 btree 中存 opdef 指针而非拷贝 opdef。在约 45K 个 op 的大型程序上：BM_CompileLarge 从 28.5s 降到 22.4s(−21.61%)。
  - Rust 备注：哈希求交集替代排序求交 = 用 `HashSet` 做 O(n) 交集；复用临时哈希表 = 复用已分配容器 (`clear()` 重用容量)；btree 对应 `BTreeMap`；存指针不拷贝 = 存 `&OpDef` 引用或 `Rc`/索引。

- **MapReduce 改进 (wordcount 基准约 2X 提速)(MapReduce improvements, ~2X speedup)** — 改造 SafeCombinerMapOutput 的 combiner 数据结构：用 `hash_map<SafeCombinerKey, ValuePtr*>`(ValuePtr 为带重复计数的值链表) 替代每条唯一键/值都占一个哈希表项的 `hash_multimap<SafeCombinerKey, StringPiece>`，从而显著降低内存 (每个值仅需 `sizeof(ValuePtr)+value_len`，减少 reducer 缓冲区的刷新次数)、显著提速 (对已存在的键新增值只需挂入链表而不新增表项)，并能以带重复计数的单一链表项表示重复值序列; 为默认的 KeyFingerprintSharding 增加 `nshards==1` 的判断以完全跳过键指纹; 将热路径上部分 VLOG(3) 改为 DVLOG(3)。使某个 wordcount 基准从 12.56s 降到 6.55s。
  - Rust 备注：多重映射 → 值链表的重构 = `HashMap<K, Vec<V>>` 或带计数的自定义结构；重复计数 = run-length 编码；DVLOG 对应仅在 debug 下启用的日志宏。

- **重写 SelectServer 中的定时器处理代码以显著提升性能 (增删一个 alarm 从 771ns 降到 281ns)(Rework SelectServer alarm handling)** — 用 `AdjustablePriorityQueue<Alarm>` 取代 `set<Alarm*>` 作为 AlarmQueue，将增删 alarm 的耗时从 771 纳秒降到 281 纳秒 (每次 alarm 设置省去一次红黑树节点的分配/释放，且堆基于 vector 实现，缓存局部性更好、每轮 selectserver 循环触碰的缓存行更少); 将 Alarmer 中的 AlarmList 从 hash_map 改为 dense_hash_map，再省一次分配/释放并改善缓存局部性; 移除 num_alarms_stat_ 与 num_closures_stat_ 这两个 MinuteTenMinuteHourStat 监控对象及对应导出变量 (即便改为 Atomic32 也仍会把耗时从 281ns 增回到 340ns)。基准 BM_AddAlarm/1 的 CPU 时间从 771ns(汇总标题写作 271ns) 降到 281ns。
  - Rust 备注：基于 vector 的二叉堆对应 `BinaryHeap`(而非红黑树式的 `BTreeSet`)，兼具更好缓存局部性并省去每节点堆分配;dense_hash_map 对应开放寻址的 `hashbrown`；监控计数即便用原子也有成本，对应权衡 `AtomicU32` 的开销。

- **索引服务速度提升 3.3 倍！(3.3X performance in index serving speed!)** — 2001 年从磁盘索引切换到内存索引时发现的一批性能问题，本次改动修复后使双路 Pentium III、2GB 内存索引的内存查询从 150 QPS 提升到 500+ QPS。改动包括：大幅提升索引块解码速度 (微基准从 8.9 MB/s 到 13.1 MB/s); 解码时对块做校验和，从而所有 getsymbol 操作都无需边界检查; 用粗糙的宏把 BitDecoder 各字段在整个循环内保存到局部变量、循环末再写回; 用内联汇编调用 Intel 的 `bsf` 指令实现 getUnary(找字中首个 1 位); 解码值到 vector 时把 resize 移出循环、只沿指针游走而非逐个做边界检查的写入;docid 解码时保持在本地 docid 空间以避免乘以 num_shards_，仅在真正需要时再换算为全局 docid;IndexBlockDecoder 导出 AdvanceToDocid 接口以按本地 docid 扫描; 位置数据按需解码而非整块急切解码; 若索引块在页边界前 4 字节内结束则复制到本地缓冲，从而始终能以 4 字节加载而不担心越过 mmap 页导致段错误; 各评分数据结构只初始化前 nterms_ 个元素而非全部 MAX_TERMS(某些情况每文档省去 20K~100K 的 memset); 当中间评分值为 0 时跳过 round_to_int 及后续计算 (最常见情形); 把评分数据结构的边界检查改为 debug 模式断言。
  - Rust 备注：校验和后省边界检查 = `get_unchecked`(`unsafe`) 或迭代器消除边界检查; 内联汇编 `bsf` 对应 `u32::trailing_zeros`(编译为 `tzcnt`/`bsf`); 把字段提到局部变量 = 帮助编译器保持在寄存器; 按需解码 = 惰性求值;debug 断言 = `debug_assert!`。

## 扩展阅读 (Further reading)

以下是作者们觉得有帮助的性能相关书籍与文章，排名不分先后：

- [Optimizing software in C++](https://www.agner.org/optimize/optimizing_cpp.pdf)，作者 Agner Fog。介绍了许多提升性能的实用底层技巧。
- [Understanding Software Dynamics](https://www.oreilly.com/library/view/understanding-software-dynamics/9780137589692/)，作者 Richard L. Sites。涵盖诊断与修复性能问题的专家方法与高级工具。
- [Performance tips of the week](https://abseil.io/fast/) —— 一系列实用小贴士的合集。
- [Performance Matters](https://travisdowns.github.io/) —— 一系列关于性能的文章。
- [Daniel Lemire 的博客](https://lemire.me/blog/) —— 有趣算法的高性能实现。
- [Building Software Systems at Google and Lessons Learned](https://www.youtube.com/watch?v=modXC5IWTJI) —— 一段视频，讲述 Google 十年间遇到的系统性能问题。
- [Programming Pearls](https://books.google.com/books/about/Programming_Pearls.html?id=kse_7qbWbjsC) 与 [More Programming Pearls: Confessions of a Coder](https://books.google.com/books/about/More_Programming_Pearls.html?id=a2AZAQAAIAAJ)，作者 Jon Bentley。关于如何从算法出发、最终得到简洁高效实现的随笔。
- [Hacker's Delight](https://en.wikipedia.org/wiki/Hacker%27s_Delight)，作者 Henry S. Warren。用于解决一些常见问题的位级与算术算法。
- [Computer Architecture: A Quantitative Approach](https://books.google.com/books/about/Computer_Architecture.html?id=cM8mDwAAQBAJ)，作者 John L. Hennessy 与 David A. Patterson —— 涵盖计算机体系结构的诸多方面，包括注重性能的软件开发者应了解的缓存、分支预测器、TLB 等内容。

Rust 侧补充：[The Rust Performance Book](https://nnethercote.github.io/perf-book/)(nnethercote)，以及 [rayon](https://docs.rs/rayon/)、[criterion](https://docs.rs/criterion/)、[hashbrown](https://docs.rs/hashbrown/)、[bumpalo](https://docs.rs/bumpalo/)、[smallvec](https://docs.rs/smallvec/) 等 crate 的文档。

## 建议引用 (Suggested citation)

如果你想引用本文档，我们建议：

```shell
Jeffrey Dean & Sanjay Ghemawat, Performance Hints, 2025, https://abseil.io/fast/hints.html
```

或使用 BibTeX：

```bibtex
@misc{DeanGhemawatPerformance2025,
  author = {Dean, Jeffrey and Ghemawat, Sanjay},
  title = {Performance Hints},
  year = {2025},
  howpublished = {\url{https://abseil.io/fast/hints.html}},
}
```

(说明：以上为原文档上游的署名与引用信息，按原样保留。)

## 致谢 (Acknowledgments)

许多同事为本文档提供了有益的反馈，包括：

- Adrian Ulrich
- Alexander Kuzmin
- Alexei Bendebury
- Alexey Alexandrov
- Amer Diwan
- Austin Sims
- Benoit Boissinot
- Brooks Moses
- Chris Kennelly
- Chris Ruemmler
- Danila Kutenin
- Darryl Gove
- David Majnemer
- Dmitry Vyukov
- Emanuel Taropa
- Felix Broberg
- Francis Birck Moreira
- Gideon Glass
- Henrik Stewenius
- Jeremy Dorfman
- John Dethridge
- Kurt Kluever
- Kyle Konrad
- Lucas Pereira
- Marc Eaddy
- Michael Marty
- Michael Whittaker
- Mircea Trofin
- Misha Brukman
- Nicolas Hillegeer
- Ranjit Mathew
- Rasmus Larsen
- Soheil Hassas Yeganeh
- Srdjan Petrovic
- Steinar H. Gunderson
- Stergios Stergiou
- Steven Timotius
- Sylvain Vignaud
- Thomas Etter
- Thomas Köppe
- Tim Chestnutt
- Todd Lipcon
- Vance Lankhaar
- Victor Costan
- Yao Zuo
- Zhou Fang
- Zuguang Yang

<!-- obsidian-publish-source: 40_Programming_SE_编程与软件工程/30_Debug_Performance_调试与性能/Performance Optimization 性能优化/Performance Hints/「译」Performance Hints (Rust 版).md -->