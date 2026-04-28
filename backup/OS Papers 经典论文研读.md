---

## 第一部分：操作系统的基石与边界重构

**1. The UNIX Time-Sharing System (SOSP 1973)**

* **底层命题：** 如何用最少的抽象原语（File, Process, Pipe）构建一个通用的多用户分时系统？
* **学习价值（设计的极简主义）：** 它是所有现代 OS 的祖师爷。你可以学到什么叫“如无必要，勿增实体”。Unix 没有过度设计复杂的 IPC，而是通过“一切皆文件”将计算和 I/O 统一。它演示了如何用极简的接口定义解决极其复杂的系统资源复用问题。

**2. The Design and Implementation of a Log-Structured File System (SOSP 1991)**

* **底层命题：** 当内存越来越大（读操作大多命中缓存），而磁盘寻道时间没有实质改善时，如何彻底消除随机写带来的性能瓶颈？
* **学习价值（硬件约束驱动的架构范式转移）：** LFS 是“识别硬件底层物理约束，并反向重构软件层”的教科书。它将整个文件系统当成一个连续的 Log 来写。你可以通过这篇论文学习如何严丝合缝地将“Problem（磁盘随机写慢）”映射到“Design（全量顺序追加写 + 垃圾回收）”，以及它是如何权衡随之而来的写放大代价的。

**3. The Multikernel: A new OS architecture for scalable multicore systems (SOSP 2009)**

* **底层命题：** 当多核处理器的核数持续增加，核间通信的延迟已经接近网络通信时，传统的共享内存 OS 架构是否还有效？
* **学习价值（打破基础假设）：** 著名的 Barrelfish 操作系统。它挑战了 Linux/Windows 最核心的“共享内存”假设，提出把单机多核当成一个分布式网络来做操作系统（Message passing over shared memory）。这篇论文会教你如何敏锐地捕捉到底层硬件拓扑的变化，并敢于推翻整个领域的系统模型。

---

## 第二部分：隔离、虚拟化与安全验证

**4. Xen and the Art of Virtualization (SOSP 2003)**

* **底层命题：** 在缺乏硬件虚拟化支持（如 Intel VT-x 尚未普及）的时代，如何以接近原生的性能同时运行多个隔离的操作系统？
* **学习价值（妥协的艺术与接口设计）：** 引入了“半虚拟化（Paravirtualization）”的概念。它没有死磕“完全透明的虚拟化”，而是选择修改 Guest OS 内核来主动适配底层 Hypervisor。你可以学到在面临严苛的性能约束时，如何通过重新定义系统边界（API/ABI）来达成最优解。

**5. seL4: Formal Verification of an OS Kernel (SOSP 2009)**

* **底层命题：** 我们能否在数学意义上证明一个通用微内核的 C 代码实现 100% 符合其高层安全规范，且不引入无法接受的性能开销？
* **学习价值（真正的 Correct-by-Construction）：** 如果你对我们刚才讨论的 SpecFS 论文中模糊的“规范生成”感到不满，seL4 就是最极致的对立面。它展示了什么叫严格的机器证明。你可以学到为了适应形式化验证，系统架构必须做出哪些彻底的让步（如资源分配推给用户态），以及系统安全模型到底该如何建立。

**6. Komodo: Using verification to disentangle secure-enclave hardware from software (SOSP 2017)**

* **底层命题：** 现有的硬件可信执行环境（如 SGX）将复杂的安全逻辑硬连线在硅片中，导致漏洞难以修复且无法验证，能否将硬件的安全监控器解耦到可形式化验证的软件层？
* **学习价值（软硬协同与机密计算）：** 对于关注系统安全和硬件级隔离的工程师来说，这篇论文深刻揭示了硬件飞地（Enclave）的底层机制，并演示了如何用软件（基于 ARM TrustZone）接管安全边界，是研究机密计算体系结构的必读之作。

---

## 第三部分：从逻辑时钟到分布式共识

**7. Spanner: Google's Globally-Distributed Database (OSDI 2012)**

* **底层命题：** 在全球范围内部署的分布式系统中，如何提供严格的外部一致性（External Consistency）而不需要跨越半个地球的集中式锁？
* **学习价值（用物理法则解决软件难题）：** 如果你已经了解了 Lamport 的逻辑时钟（Time, Clocks, and the Ordering of Events），Spanner 就是它的现代工业级升华。Google 没有继续在纯软件逻辑上绕圈子，而是引入了 GPS 和原子钟（TrueTime API），将时间的不确定性量化为一个绝对的误差边界 $[t.earliest, t.latest]$。这是系统设计中“用硬件创新降维打击软件复杂度”的巅峰。

---

## 第四部分：语言安全与新型系统架构

**8. Redleaf: Isolation and Communication in a Safe Operating System (OSDI 2020)**

* **底层命题：** 如果我们使用一种内存安全的语言（Rust），能否抛弃昂贵的硬件页表隔离（MMU），转而利用语言的类型系统来实现轻量级、零拷贝的内核模块隔离？

* **学习价值（Rust 进内核的哲学）：** 这不是一篇简单地“用 Rust 重写 C”的论文。它深度利用了 Rust 的所有权（Ownership）和线性类型（Linear Types）特性，重新定义了 IPC（进程间通信）和故障恢复机制。它会让你深刻理解，新的语言特性如何直接改变操作系统的底层架构选型。

[[RedLeaf 一个安全操作系统中的隔离与通信]]

---

## 第五部分：AI 基础设施与底层优化的碰撞

**9. vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention (SOSP 2023)**

* **底层命题：** 为什么大语言模型在推理时显存利用率如此低下？根本原因是由于 KV Cache 的动态增长导致了严重的内存碎片。
* **学习价值（跨界思维）：** 这篇论文极其优美地将操作系统中最古老的概念——**虚拟内存分页（Paging）**——应用到了 AI 大模型的显存管理中。这是一个典型的“问题定义极其精准，设计完全对应问题”的高水平论文。它会教你如何用经典的 OS 思想去解决最前沿的 AI 系统瓶颈。

**10. MapReduce: Simplified Data Processing on Large Clusters (OSDI 2004)**

* **底层命题：** 如何让完全不懂分布式系统、容错、网络调度的业务工程师，也能在成千上万台机器上编写并处理海量数据？
* **学习价值（抽象的威力）：** 将一切复杂的分布式计算抽象为 `Map` 和 `Reduce` 两个接口。尽管如今它的实现已被 Spark/Flink 取代，但它在“如何划定系统框架与用户逻辑的边界”、“如何通过设计彻底屏蔽底层硬件故障”上的第一性原理思考，至今无法被超越。

-----

> 这几年看的 paper 不少，但我觉得真正矫正我 first principle 的还是真就那几个 [Turing Award](https://zhida.zhihu.com/search?content_id=766385283&content_type=Answer&match_order=1&q=Turing+Award&zhida_source=entity) 的老 paper，Dijkstra 和 Knuth 对编程的思考; Lamport 的 existence of refinement mapping, Codd 的 relational model, Jim Gray 的 transactions 等等。人就是很微妙，看了很多东西看不懂，觉得懂了就变成 small explanation 了，也就是“压缩即智能”吧。

---

## 1. Edsger W. Dijkstra: 编程的艺术与逻辑

Dijkstra 不仅仅发明了算法，他更确立了“编程是数学分支”的思想。

* **《Go To Statement Considered Harmful》 (1968)**：这不仅是关于一个语法的争论，它确立了**结构化编程**的核心——程序的静态结构（代码）与动态执行过程之间的对应关系必须是可预测的。
* **《Notes on Structured Programming》 (1970)**：他在这里系统地讨论了如何通过分层抽象和严格的逻辑推导来处理软件的复杂性。
* **《A Discipline of Programming》 (1976)**：引入了著名的 **GCL (Guarded Command Language)**，通过“弱前置条件”推导程序正确性。

## 2. Donald Knuth: 算法的严谨性

Knuth 矫正的是我们对“程序效率”和“文学化表达”的认知。

* **《The Art of Computer Programming》 (TAOCP)** 系列：虽然是书，但他早期的论文（如关于 LR 解析器的研究）奠定了基础。他教导我们：只有通过对算法复杂度的极致分析（即他的 $O$ 符号体系），才能真正理解计算的边界。
* **《Literate Programming》 (1984)**：他认为程序应该像文学作品一样阅读，代码是写给人看的，顺便给机器执行。

## 3. Leslie Lamport: 分布式系统的真理

Lamport 的工作是所有分布式共识和状态机理论的源头。

* **《The Existence of Refinement Mappings》 (1991)**：这是你专门提到的那一篇。它解决了“如何证明一个高层规范（Specification）与底层实现（Implementation）是等价的”这一哲学问题。它通过 **TLA+**（时序逻辑操作）的思想，定义了什么是“正确性步进”。
* **《Time, Clocks, and the Ordering of Events in a Distributed System》 (1978)**：分布式领域的圣经，定义了偏序关系（Happened-before）和逻辑时钟。

## 4. Edgar F. Codd: 关系模型的奠基

Codd 将数据从繁琐的物理存储逻辑（如指针、链表）中解放出来。

* **《A Relational Model of Data for Large Shared Data Banks》 (1970)**：这篇论文提出了“关系论”代替“层次模型”。它强制我们将逻辑表示与物理实现分离，这是数据库领域最伟大的“第一性原理”。

## 5. Jim Gray: 事务的鲁棒性

Jim Gray 定义了现代系统的“可靠性”语义。

* **《Notes on Data Base Operating Systems》 (1978)**：在这里他系统化了 **ACID** 的雏形，特别是原子性（Atomicity）和两阶段提交（2PC）。
* **《The Transaction Concept: Virtues and Limitations》 (1981)**：深入探讨了事务作为处理并发和故障的通用抽象，这是所有高并发系统的基石。

---

这些材料通常不关注具体的工程实现（Implementation），而是关注**表达能力的边界**、**系统复杂度的管理**以及**软硬件的契约**。

---

## 1. 表达能力与计算本质：Alan Turing & Alonzo Church

如果你想理解为什么 Rust 的类型系统能做这么多事，或者为什么函数式编程不是异类，必须回到源头。

* **Alan Turing: 《On Computable Numbers, with an Application to the Entscheidungsproblem》 (1936)**
* **第一性原理：** 它定义了什么是“可计算”。它告诉我们，无论现在的量子计算机或 AI 多么强大，只要它符合图灵机模型，其逻辑边界在 1936 年就被锁死了。

* **John Backus: 《Can Programming Be Liberated from the von Neumann Style?》 (1977)**
* **第一性原理：** 图灵奖演说。他挑战了指令式编程的本质（冯·诺依曼瓶颈），提出了函数式编程作为一种更纯粹的数学表达方式。这对理解现代语言中的不变性（Immutability）和声明式编程至关重要。

## 2. 系统设计的“奥卡姆剃刀”：Saltzer & Schroeder

在安全和系统架构领域，这两位制定的原则至今没有被超越。

* **Jerome Saltzer & Michael Schroeder: 《The Protection of Information in Computer Systems》 (1975)**

* **第一性原理：** 提出了**最小特权原则 (Least Privilege)**、**经济机制 (Economy of Mechanism)** 等 8 大原则。
* **为什么重要：** 你正在研究的 TEE（可信执行环境）或 Confidential Computing，其核心诉求——“减少 TCB (Trusted Computing Base)”，本质上就是这篇论文中“Economy of Mechanism”的极致体现。

## 3. 操作系统与并发的元模型：Brinch Hansen & Hoare

如果你对多线程并发感到痛苦，那是由于你还没看到这些“元抽象”。

* **C.A.R. Hoare: 《Communicating Sequential Processes》 (CSP, 1978)**
* **第一性原理：** “不要通过共享内存来通信，而要通过通信来共享内存”。这是 Go 语言和 Rust 某些并发模型的哲学底座。

* **Per Brinch Hansen: 《Monitors: An Operating System Structuring Concept》 (1974)**
* **第一性原理：** 定义了管程（Monitor），这是几乎所有现代高级语言（Java, C++ 等）处理并发锁的逻辑模型。

## 4. 复杂性管理的哲学：Brooks & Parnas

这两位教导我们如何处理软件工程中那些“看不见”的熵。

* **Fred Brooks: 《No Silver Bullet — Essence and Accident in Software Engineering》 (1986)**
* **第一性原理：** 区分了**本质复杂度 (Essence)** 和 **偶然复杂度 (Accident)**。
* **认知压缩：** 它让你在看到一个新技术时，瞬间判断出它是在解决问题本身（本质），还是只是在优化工具（偶然）。

* **David Parnas: 《On the Criteria to Be Used in Decomposing Systems into Modules》 (1972)**
* **第一性原理：** 提出了**信息隐藏 (Information Hiding)**。这不是简单的“私有变量”，而是关于“变化的隔离”。

<!-- obsidian-publish-source: Operating System 操作系统/Pappers 论文/OS Papers 经典论文研读.md -->