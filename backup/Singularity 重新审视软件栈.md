**Galen C. Hunt 与 James R. Larus**
Microsoft Research Redmond
galenh@microsoft.com

---

## 摘要

每一个操作系统都体现了一系列设计决策。当今最流行的操作系统背后的许多决策始终未曾改变，尽管硬件和软件已经发生了巨大的演进。操作系统构成了几乎所有软件栈的基础，因此现有系统中的不足具有广泛而深远的影响。本文描述了 Singularity 项目在编程语言和验证工具取得进展的背景下，重新审视这些设计决策所做的努力。

Singularity 系统包含三个关键的架构特征：用于保护程序和系统服务的软件隔离进程（Software-Isolated Process, SIP）、用于通信的基于契约的通道（Contract-Based Channel）、以及用于系统属性验证的基于清单的程序（Manifest-Based Program, MBP）。我们将详细描述这一基础，并概述在此基础之上构建的实验性系统中正在进行的研究。

**关键词：** 操作系统、安全编程语言、程序验证、程序规约、密封进程架构、密封内核、软件隔离进程（SIP）、硬件保护域、基于清单的程序（MBP）、不安全代码税。

---

## 1. 引言

每一个操作系统都体现了一系列设计决策——有些是显式的，有些是隐式的。这些决策涵盖了实现语言的选择、程序保护模型、安全模型、系统抽象以及诸多其他方面。

当代操作系统——Windows、Linux、Mac OS X 和 BSD——共享了大量的设计决策。这种共性并非完全出于偶然，因为这些系统均植根于 20 世纪 60 年代末和 70 年代初的操作系统架构与开发工具。鉴于相同的运行环境、相同的编程语言以及相似的用户期望，这些系统的设计者做出了类似的决策也就不足为奇了。虽然某些设计决策经受住了时间的考验，但另一些则随着时间的推移而逐渐暴露出不足。

Singularity 项目始于 2003 年，旨在重新审视现有系统和软件栈的设计决策及其日益明显的缺陷。这些缺陷包括：广泛存在的安全漏洞、应用程序之间出乎意料的交互、由错误的扩展程序（extension）、插件（plug-in）和驱动程序（driver）导致的故障，以及系统鲁棒性不足的普遍感受。

我们认为，这些问题中的许多可归因于系统尚未从 20 世纪 60 年代和 70 年代的计算机架构和编程语言中充分演进。那个时期的计算环境与今天截然不同。计算机在速度和内存容量方面极为有限。它们仅被一小部分善意的技术精英所使用，且很少联网或与物理设备连接。这些前提条件已无一成立，但现代操作系统并未随之演进以适应计算机使用方式的巨大变迁。

### 1.1 一段旅程，而非终点

在 Singularity 项目中，我们构建了一个新的操作系统、一种新的编程语言和新的软件验证工具。Singularity 操作系统采用了一种基于进程软件隔离的全新软件架构。我们的编程语言 Sing# [8] 是 C# 的一个扩展，为操作系统通信原语提供可验证的一等支持（first-class support），同时为系统编程和代码分解（code factoring）提供强有力的支持。健全的验证工具能够在开发周期的早期阶段检测程序员的错误。

从一开始，Singularity 就受到以下问题的驱动：如果一个软件平台从零开始设计，以提高可靠性（dependability）和可信赖性（trustworthiness）为首要目标，它将呈现怎样的面貌？为此，我们倡导了三项策略。第一，广泛使用安全编程语言消除了许多可预防的缺陷，例如缓冲区溢出（buffer overrun）。第二，使用健全的程序验证工具进一步保证在开发周期的早期阶段移除整类程序员错误。第三，改进的系统架构在明确定义的边界处阻止运行时错误的传播，使得实现鲁棒且正确的系统行为变得更加容易。

尽管在研究原型中难以衡量可靠性，但我们的经验使我们确信新技术和设计决策的实用性，我们相信这些将在未来带来更加鲁棒和可靠的系统。

Singularity 是一个用于实验新设计思想的实验室，而非一种设计方案。虽然我们乐于认为当前的代码库相比先前工作代表了一个重大的进步，但我们并不将其视为一个 " 理想 " 的系统或自身的终极目标。诸如 Singularity 这样的研究原型有意保持为一个进行中的工作（work in progress）；它是一个实验室，我们在其中持续探索各种实现方案与权衡取舍。

在本文的其余部分，我们将描述所有 Singularity 系统共享的通用架构基础。第 3 节描述了 Singularity 内核的实现，它提供了该基础的底层实现。第 4 节概述了我们在过去三年中在 Singularity 项目内所开展的工作，以探索操作系统和系统设计空间中的新机遇。最后，在第 5 节中，我们总结了迄今为止的工作并讨论了未来的研究方向。

---

## 2. 架构基础

Singularity 系统由三个关键的架构特征组成：软件隔离进程（Software-Isolated Process, SIP）、基于契约的通道（Contract-Based Channel）和基于清单的程序（Manifest-Based Program, MBP）。软件隔离进程为程序执行提供了一个免受外部干扰的环境。基于契约的通道实现了进程之间快速、可验证的基于消息的通信。基于清单的程序定义了在软件隔离进程中运行的代码，并规约其可验证的行为属性。

Singularity 背后的一个指导哲学是简洁优于丰富（simplicity over richness）。上述基础特征分别为执行、通信和系统验证提供基本支持，并构成一个坚实的设计基石，在此之上可以构建可靠的、可验证的系统。

### 2.1 软件隔离进程

Singularity 系统的第一个基础特征是软件隔离进程（Software-Isolated Process, SIP）。与许多操作系统中的进程类似，SIP 是处理资源的持有者，并为程序执行提供上下文环境。然而，与传统进程不同的是，SIP 利用现代编程语言的类型安全（type safety）和内存安全（memory safety）来大幅降低隔离安全代码的开销。

图 1 展示了 SIP 的架构特征。SIP 与其他操作系统中的进程共享许多属性。每个用户程序的执行都在 SIP 的上下文中进行。与 SIP 关联的是一组包含代码和数据的内存页。一个 SIP 包含一个或多个执行线程。SIP 以一个安全身份执行，并具有关联的操作系统安全属性。最后，SIP 提供信息隐藏（information hiding）和故障隔离（failure isolation）。

SIP 的某些方面在先前的操作系统中已有探索，但不如 Singularity 中那样严格。SIP 不共享数据，因此所有通信都通过在称为通道（channel）的消息传递管道上交换消息来进行。Singularity 增加了静态可验证契约的严格性。契约（contract）规约了给定类型的所有通道上通信的消息和协议。SIP 通过向内核的应用程序二进制接口（Application Binary Interface, ABI）访问原始功能，例如发送和接收消息等。Singularity ABI 具有严格的设计，包含完全声明式的版本信息。它提供对计算最基本方面——内存、执行和通信——的安全本地访问，并排除了语义含糊的构造，例如 `ioctl`。

SIP 在其他方面也不同于传统操作系统进程。它们不能与其他 SIP 共享可写内存；SIP 中的代码在执行时是密封的（sealed）；SIP 通过软件验证而非硬件保护实现隔离。

换言之，SIP 是封闭的对象空间（closed object space）。一个 SIP 中的对象不能被任何其他 SIP 修改或直接访问。SIP 之间的通信通过在消息中转移数据的独占所有权（exclusive ownership）来进行。一个线性类型系统（linear type system）和一块被称为交换堆（exchange heap）的特殊内存区域允许轻量级地交换甚至非常大量的数据，但不允许共享。

因此，SIP 自主执行：每个 SIP 拥有自己的数据布局、运行时系统和垃圾回收器（garbage collector）。

SIP 是密封的代码空间（sealed code space）。作为密封进程，SIP 不能动态加载或生成代码到自身中。密封进程为操作系统和应用程序提供了一种通用的扩展机制：扩展代码在一个新进程中执行，与其宿主的 SIP 相分离。密封进程提供了多项优势。它们增强了程序分析工具静态检测缺陷的能力。它们使得更强的安全机制成为可能，例如通过代码内容来标识一个进程。它们还可以消除在运行时执行环境中复制操作系统级访问控制的需要。最近的一篇 Singularity 论文 [14] 对密封进程的权衡和收益进行了全面分析。

SIP 依赖编程语言的类型和内存安全来实现隔离，而不是依赖内存管理硬件。通过静态验证和运行时检查的组合，Singularity 验证 SIP 中的用户代码不能访问 SIP 之外的内存区域。由于进程隔离由软件而非硬件提供，多个 SIP 可以驻留在单一的物理或虚拟地址空间中。在最简单的 Singularity 系统中，内核和所有 SIP 共享同一个内核模式地址空间。如第 4.2 节所讨论的，通过将 SIP 分层到用户和内核保护级别的多个地址空间中，可以构建更丰富的系统。Aiken 等人 [1] 评估了软件隔离和硬件隔离之间的权衡。

SIP 之间的通信开销很低，且 SIP 的创建和销毁成本也很低廉，相较于硬件保护进程而言（见表 1）。与硬件保护进程不同，创建 SIP 无需创建页表（page table）或刷新 TLB（Translation Lookaside Buffer，转译后备缓冲区）。SIP 之间的上下文切换也具有非常低的开销，因为 TLB 和虚拟寻址的缓存无需刷新。在终止时，SIP 的资源可以被高效回收，其内存页可以在不涉及垃圾回收的情况下被回收利用。

**表 1. 基本进程开销（以 CPU 周期计），在 AMD Athlon 64 3000+（1.8 GHz）CPU、NVIDIA nForce4 Ultra 芯片组上测量。**

| 开销（CPU 周期） | API 调用 | 线程让出 | 消息 Ping/Pong | 进程创建 |
|---|---|---|---|---|
| Singularity | 80 | 365 | 1,040 | 388,000 |
| FreeBSD | 878 | 911 | 13,300 | 1,030,000 |
| Linux | 437 | 906 | 5,800 | 719,000 |
| Windows | 627 | 753 | 6,340 | 5,380,000 |

低成本使得将 SIP 用作细粒度的隔离和扩展机制成为实际可行的方案，以替代传统的硬件保护进程和不受保护的动态代码加载机制。因此，Singularity 只需要一种错误恢复模型、一种通信机制、一种安全架构和一种编程模型，而不是当前系统中那些层层叠叠、部分冗余的机制和策略。

Singularity 项目中的一个关键实验是完全使用 SIP 来构建整个操作系统，并证明由此产生的系统比传统系统更加可靠。迄今为止的结果是令人鼓舞的。SIP 足够轻量，能够适配 " 自然 " 的软件开发粒度——即每个 SIP 对应一个开发者或团队——并且足够轻量级以提供对异常行为的故障停止（fault-stop）边界。

### 2.2 基于契约的通道

Singularity 中 SIP 之间的所有通信都通过基于契约的通道（contract-based channel）进行。通道是一个双向的消息管道，恰好有两个端点（endpoint）。通道提供无损的、有序的消息队列。从语义上讲，每个端点都有一个接收队列。在一个端点上发送消息会将消息放入另一个端点的接收队列中。通道端点在任何时刻恰好属于一个线程。只有端点的所属线程才能从其接收队列中取出消息或向其对端发送消息。

通道上的通信由通道契约（channel contract）来描述。通道的两端在契约中是不对称的。一个端点是导入端（Imp），另一个是导出端（Exp）。在 Sing# 语言中，端点分别由类型 `C.Imp` 和 `C.Exp` 来区分，其中 C 是控制交互的通道契约。

通道契约在 Sing# 语言中声明。契约由消息声明和一组命名的协议状态组成。消息声明说明了每条消息的参数数目和类型以及可选的消息方向。每个状态指定了通向其他状态的可能消息序列。

我们将通过清单 1 中所示的网络设备驱动程序契约的精简版来解释通道契约。通道契约是从导出服务的 SIP 的角度编写的，并从第一个列出的状态开始。消息序列由消息标签和消息方向符号组成（`!` 表示 Exp 到 Imp，`?` 表示 Imp 到 Exp）。如果消息声明已经包含方向（`in`、`out`），则方向符号并非严格必要，但符号使状态机更易于人类阅读。

在我们的例子中，第一个状态是 `START`，网络设备驱动程序通过发送 `DeviceInfo` 消息向客户端（可能是网络栈）传送设备信息来开始对话。之后对话进入 `IO_CONFIGURE_BEGIN` 状态，客户端必须发送 `RegisterForEvents` 消息以注册另一个通道用于接收事件，并设置各种参数以将对话推进到 `IO_CONFIGURED` 状态。如果参数设置出错，驱动程序可以通过发送 `InvalidParameters` 消息强制客户端重新开始配置。

一旦对话进入 `IO_CONFIGURED` 状态，客户端可以启动 I/O（通过发送 `StartIO`），或重新配置驱动程序（`ConfigureIO`）。如果 I/O 被启动，对话进入 `IO_RUNNING` 状态。在 `IO_RUNNING` 状态下，客户端向驱动程序提供用于接收数据包的缓冲区（消息 `PacketForReceive`）。驱动程序可能以 `BadPacketSize` 回应，返回缓冲区并指示所期望的大小。这样，客户端可以为传入的数据包提供多个缓冲区。客户端可以通过 `GetReceivedPacket` 消息请求已接收数据的数据包，驱动程序要么通过 `ReceivedPacket` 返回该数据包，要么声明没有更多包含数据的数据包（`NoPacket`）。传输数据包的类似消息序列也存在，但我们在示例中省略了它们。

从通道契约来看，客户端似乎是轮询驱动程序以获取数据包。然而，我们尚未解释 `RegisterForEvents` 消息的参数。类型为 `NicEvents.Exp:READY` 的参数描述了 `NicEvents` 契约在 `READY` 状态下的一个 Exp 通道端点。该端点参数在客户端和网络驱动程序之间建立了第二条通道，驱动程序通过该通道通知客户端重要事件（例如数据包突发到达的开始）。客户端在准备就绪时通过 `NicDevice` 通道获取数据包。图 2 显示了客户端和网络驱动程序之间的通道配置。`NicEvents` 契约如清单 2 所示。

通道实现了 SIP 之间高效且可分析的通信。结合对线性类型的支持，Singularity 允许在 SIP 之间零拷贝（zero-copy）地交换大量数据 [8]。此外，Sing# 编译器可以静态验证通道上的发送和接收操作永远不会在错误的协议状态下被执行。一个独立的契约验证器可以读取程序的字节码并静态验证程序中使用了哪些契约，以及代码是否符合契约协议中描述的状态机。

经验表明，通道契约是防止和检测错误的宝贵工具。团队中的程序员最初对契约的价值持怀疑态度。契约一致性验证器直到我们首次实现网络栈和 Web 服务器将近一年之后才完成。当验证器上线时，它立即标记了网络栈和 Web 服务器交互中的一个错误。该错误出现在程序员未能预料到传入套接字（socket）缺少数据的情况，该情况由 `NO_DATA` 消息表达。这个 bug 在 Web 服务器中存在了将近一年。验证器在几秒钟内就标记了该错误，并确定了触发该 bug 的确切条件。

通道契约在交互组件之间提供了清晰的关注点分离（separation of concerns），并有助于在高层次上理解系统架构。静态检查帮助程序员避免运行时的 " 消息不被理解 " 错误。此外，通道的运行时语义将故障的可观察性限制为仅在接收操作上，从而消除了在发送操作处处理错误条件的需要——这在发送处是不方便的。

### 2.3 基于清单的程序

Singularity 系统的第三个基础架构特征是基于清单的程序（Manifest-Based Program, MBP）。MBP 是由一个静态清单（manifest）定义的程序。没有清单，任何代码都不允许在 Singularity 系统上运行。实际上，要启动执行，用户调用的是一个清单，而不是像其他系统中那样的可执行文件。

清单描述了 MBP 的代码资源、所需的系统资源、期望的能力以及对其他程序的依赖关系。当 MBP 被安装到系统上时，清单被用来识别和验证 MBP 的代码是否满足所有必需的安全属性，确保 MBP 的所有系统依赖都能被满足，并防止安装 MBP 对任何先前已安装 MBP 的执行产生干扰。在执行之前，清单被用来发现影响 MBP 的配置参数以及对这些配置参数的限制。当 MBP 被调用时，清单指导将代码放置到 SIP 中执行、在新 SIP 和其他 SIP 之间连接通道、以及由 SIP 授予对系统资源的访问。

清单不仅仅是对程序的描述或 SIP 代码内容的清单；它是 MBP 预期行为的机器可检查的声明式表达。清单的主要目的是允许对 MBP 的属性进行静态和动态验证。例如，设备驱动程序的清单提供了足够的信息，以便在安装时验证该驱动程序不会访问由先前已安装的设备驱动程序使用的硬件。由 Singularity 验证的其他 MBP 属性包括类型和内存安全、不包含特权模式指令、对通道契约的符合性、仅使用已声明的通道契约、以及正确版本的 ABI 使用。

MBP 的代码可以作为清单的内联元素包含，也可以在单独的文件中提供。解释型脚本语言（如 Singularity shell 语言）可以很容易地作为清单的内联元素包含。另一方面，大型编译应用程序可能由许多二进制文件组成，一些是 MBP 独有的，一些与其他 MBP 共享并存储在与 MBP 清单分离的仓库中。

Singularity 的通用 MBP 清单可以通过内联方式或其他文件中的元数据进行扩展，以允许验证复杂的属性。例如，基本清单不足以验证 MBP 是类型安全的或它仅使用特定的通道契约子集。验证编译代码的安全性需要 MBP 二进制文件中的附加元数据。

为了尽可能多地静态验证运行时属性，Singularity MBP 的代码以编译后的 Microsoft Intermediate Language（MSIL，微软中间语言）二进制文件的形式交付到系统。MSIL 是 Microsoft Common Language Runtime（CLR，公共语言运行时）[7] 所接受的 CPU 无关的指令集。Singularity 使用标准的 MSIL 格式，Singularity 特有的功能通过 MSIL 元数据扩展来表达。除少数例外情况外，操作系统在安装时将 MSIL 编译为本机指令集。用安装时编译（install-time compilation）替代即时编译（JIT compilation）得益于清单的支持，因为清单声明了从 MBP 创建的 SIP 所需的所有可执行 MSIL 代码。

Singularity 中的每个组件都由清单描述，包括内核、设备驱动程序和用户应用程序。我们已经证明 MBP 在创建具有可验证硬件访问属性的 " 自描述 " 设备驱动程序方面特别有用 [25]。Singularity 系统使用清单特性将命令行处理从几乎所有应用程序中移出，并将其集中到 shell 程序中。清单还指导 MBP 的安装时编译。我们相信 MBP 将在显著降低系统管理成本方面发挥作用，但我们尚未完成验证这一假设的实验工作。

---

## 3. Singularity 内核

支撑 Singularity 架构基础的是 Singularity 内核。内核提供了软件隔离进程、基于契约的通道和基于清单的程序这些基础抽象。内核执行将系统资源在竞争的程序之间分配以及抽象系统硬件复杂性的关键角色。对每个 SIP，内核提供一个纯净的执行环境，包括线程、内存和通过通道访问其他 MBP 的能力。

与先前的 Cedar [26] 和 Spin [4] 项目类似，Singularity 项目享有用类型安全、垃圾回收语言编写内核带来的安全性和生产力优势。按代码行数计算，Singularity 内核中超过 90% 是用 Sing# 编写的。虽然内核的大部分是类型安全的 Sing#，但内核代码中有相当一部分是用该语言的不安全变体编写的。最重要的不安全代码是垃圾回收器，它占 Singularity 中不安全代码的 48%。其他主要的不安全 Sing# 代码来源包括内存管理和 I/O 访问子系统。Singularity 在与用 C 或 C++ 编写的内核相同的位置包含少量汇编语言代码，例如线程上下文切换、中断向量等。Singularity 内核中大约 6% 是用 C++ 编写的，主要由内核调试器和底层系统初始化代码组成。

Singularity 内核是一个微内核（microkernel）；所有设备驱动程序、网络协议、文件系统和用户可替换的服务都在内核外部的 SIP 中执行。留在内核中的功能包括调度（scheduling）、调解对硬件资源的特权访问、管理系统内存、管理线程和线程栈、以及创建和销毁 SIP 和通道。

### 3.1 ABI

SIP 通过 Singularity 应用程序二进制接口（ABI）访问原始内核功能，例如发送和接收消息的能力。Singularity ABI 遵循最小特权原则（principle of least privilege）[23]。默认情况下，SIP 除了操作自身状态以及停止和启动子 SIP 之外，什么也做不了。ABI 确保了每个 SIP 拥有一个最小化、安全且隔离的计算环境。

SIP 通过通道而非 ABI 函数来获得对更高级系统服务的访问，例如访问文件或收发网络数据包的能力。在 Singularity 中，通道端点就是能力（capability）[19, 24]。例如，SIP 只有在从另一个 SIP 接收到文件系统的端点后才能访问文件。通道端点要么在进程启动时就已存在（如基于清单的配置所规约，见第 4.1.1 节），要么通过现有通道上的消息到达。

ABI 在设计上区分了访问原始、进程局部操作的机制和访问高级服务的机制。这种区分支持将通道端点用作能力，并支持通道属性的验证。在动态方面，ABI 设计将新端点——即新的能力——进入 SIP 的途径限制为显式通道，这些通道具有规约传输的契约。静态分析工具可以使用类型信息来确定代码所访问的能力。例如，一个静态工具可以确定某个应用程序插件不可能发起分布式拒绝服务攻击，因为它不包含使用网络通道契约的代码。

内核 ABI 具有强版本控制。每个 MBP 的清单明确标识了它所需的 ABI 版本。在语言层面，程序代码针对一个特定的 ABI 接口程序集进行编译，该程序集位于一个显式命名版本的命名空间中，例如 `Microsoft.Singularity.V1.Threads`。一个 Singularity 内核可以导出其 ABI 的多个版本，以提供清晰的向后兼容路径。

表 2 提供了 Singularity ABI 函数按功能的细分。对其他微内核设计有研究的读者可能会发现 ABI 函数的数量——192 个——大得惊人。ABI 设计比这个数字所暗示的要简单得多，因为其设计倾向于显式语义而非最少的函数入口点。特别是，Singularity 不包含像 UNIX 的 `ioctl` 或 Windows 的 `CreateFile` 这样语义复杂的函数。

**表 2. Singularity ABI 函数按功能分类的数量。**

| 功能 | 函数数 |
|---|---|
| 通道（Channels） | 22 |
| 子进程（Child Processes） | 21 |
| 配置（Configuration） | 25 |
| 调试与诊断（Debugging & Diagnostics） | 31 |
| 交换堆（Exchange Heap） | 8 |
| 硬件访问（Hardware Access） | 16 |
| 链式栈（Linked Stacks） | 6 |
| 分页内存（Paged Memory） | 17 |
| 安全主体（Security Principals） | 3 |
| 线程与同步（Threads & Synchronization） | 43 |
| **合计** | **192** |

ABI 维护一个系统范围的状态隔离不变量（state isolation invariant）：一个进程不能通过 ABI 函数直接改变另一个进程的状态。对象引用不能通过 ABI 传递到内核或另一个 SIP。因此，内核和每个进程的垃圾回收器独立执行。ABI 调用仅影响调用进程的状态。例如，进程内的同步对象——如互斥锁（mutex）——不能跨 SIP 边界访问。状态隔离确保了 Singularity 进程对其状态拥有唯一控制权。

由于 SIP 依赖软件隔离而非硬件保护，跨越 SIP 和内核代码之间的 ABI 边界可以非常高效。最佳情况是 SIP 在内核硬件特权级别（x86 架构的 ring 0）下运行，与内核在同一地址空间中。然而，ABI 调用比函数调用更昂贵，因为它们必须标记 SIP 的垃圾回收空间与内核的垃圾回收空间之间的转换。

#### 3.1.1 特权代码

在任何操作系统中，系统完整性通过守护对特权指令的访问来保护。例如，加载新页表可以颠覆对进程内存的硬件保护，因此 Windows 或 Linux 等操作系统会守护对加载页表的代码路径的访问。在依赖硬件保护的操作系统中，包含特权指令的函数必须在内核中执行，内核运行在与用户代码不同的特权级别。

SIP 在特权指令的放置方面允许更大的灵活性。因为类型和内存安全保证了函数的执行完整性，Singularity 可以将带有安全检查的特权指令放置在 SIP 内部运行的可信函数中。例如，用于访问 I/O 硬件的特权指令可以在安装时安全地内联到设备驱动程序中。其他 ABI 函数也可以在安装时内联到 SIP 代码中。Singularity 利用这种安全内联来优化通道通信以及 SIP 中语言运行时和垃圾回收器的性能。

#### 3.1.2 句柄表

虽然设计禁止跨 ABI 的对象引用，但 SIP 代码有必要在内核中命名抽象对象，例如互斥锁或线程。抽象内核对象通过强类型的不透明句柄（opaque handle）暴露给 SIP，这些句柄引用内核句柄表中的槽位。在内核内部，句柄表槽位包含对字面内核对象的引用。强类型防止 SIP 代码修改或伪造非零句柄。此外，句柄表中的槽位仅在 SIP 终止时才被回收，以防止 SIP 释放一个互斥锁后仍保留其句柄并用它来操作另一个 SIP 的对象。Singularity 在 SIP 内部重用表项，因为在这种情况下保留句柄只会影响犯错的 SIP 本身。

### 3.2 内存管理

在大多数 Singularity 系统中，内核和所有 SIP 驻留在同一个由软件隔离保护的地址空间中。地址空间在逻辑上被划分为一个内核对象空间、每个 SIP 一个对象空间、以及用于通道数据通信的交换堆。一个贯穿始终的设计决策是内存独立性不变量（memory independence invariant）：指针必须指向 SIP 自身的内存或 SIP 在交换堆中拥有的内存。任何 SIP 都不能拥有指向另一个 SIP 对象的指针。此不变量确保了每个 SIP 可以在无需其他 SIP 协作的情况下进行垃圾回收和终止。

SIP 通过 ABI 调用向内核的页管理器获取内存，后者返回新的、非共享的页。这些页无需与 SIP 先前分配的内存页相邻，因为垃圾回收器不要求连续内存，尽管可以为大型对象或数组分配连续页块。除了用于 SIP 代码和堆数据的内存之外，SIP 每个线程拥有一个栈，并可以访问交换堆。

#### 3.2.1 交换堆

SIP 之间传递的所有数据必须驻留在交换堆中。图 3 显示了进程堆和交换堆之间的关系。SIP 可以包含指向自己堆和交换堆的指针。交换堆仅持有指向交换堆自身的指针（不指向进程或内核）。在系统执行期间的任何时刻，交换堆中的每个内存块最多由一个 SIP 拥有（可访问）。请注意，SIP 可能持有指向交换堆的悬空指针（dangling pointer）——即指向 SIP 不再拥有的块的指针。静态验证确保 SIP 永远不会通过悬空指针访问内存。

为了支持对此属性的静态验证，我们强制执行一个更强的属性，即 SIP 在其执行的任何时刻最多只有一个指向某个块的指针（一种称为线性性（linearity）的属性）。块的所有权只能通过在通道上的消息中发送来转移到另一个 SIP。Singularity 确保 SIP 在将块发送到消息后不再访问该块。

交换堆中每个块在任何时刻只被单一线程访问的事实也提供了一个有用的互斥（mutual exclusion）保证。此外，块的释放是静态强制执行的。在进程突然终止时，交换堆中的块通过引用计数（reference counting）来回收。块的所有权被记录，以便在 SIP 终止时释放所有相关的块。

### 3.3 线程

Singularity 内核和 SIP 是多线程的。所有线程都是内核线程（kernel thread），对内核调度器可见，调度器协调阻塞操作。在大多数 Singularity 系统中，内核线程上下文切换的性能接近用户线程的预期性能，因为对于运行在内核地址空间和内核硬件特权级别中的 SIP，不需要保护模式转换。

#### 3.3.1 链式栈

Singularity 使用链式栈（linked stack）来减少线程内存开销。这些栈按需增长，通过添加 4KB 或更大的非连续段来扩展。Singularity 的编译器执行静态过程间分析（static interprocedural analysis）以优化溢出检测的放置 [28]。在函数调用时，如果栈空间不足，SIP 代码调用一个 ABI 来分配一个新的栈段，并在该段中初始化第一个栈帧——位于正在运行的过程和其被调用者之间——以调用段解除链接（segment unlink）例程，当栈帧被弹出时将释放该段。

对于在 x86 的 ring 0 下运行的 SIP，当前栈段必须始终留出足够的空间供处理器保存中断或异常帧，之后处理程序切换到专用的每处理器中断栈。

#### 3.3.2 调度器

标准的 Singularity 调度器针对大量频繁通信的线程进行了优化。调度器维护两个可运行线程列表。第一个称为解除阻塞列表（unblocked list），包含最近变为可运行的线程。第二个称为被抢占列表（preempted list），包含被抢占的可运行线程。在选择下一个要运行的线程时，调度器按 FIFO 顺序从解除阻塞列表中移除线程。当解除阻塞列表为空时，调度器从被抢占列表中移除下一个线程（同样按 FIFO 顺序）。每当调度定时器中断发生时，解除阻塞列表中的所有线程被移到被抢占列表的末尾，其后是定时器触发时正在运行的线程。然后，从解除阻塞列表中调度第一个线程，并重置调度定时器。

这种双列表调度策略有利于以下模式的线程：被消息唤醒，执行少量工作，向其他 SIP 发送一条或多条消息，然后阻塞等待消息。这是运行消息处理循环的线程的常见行为。为了避免昂贵的调度硬件定时器重置，解除阻塞列表中的线程继承唤醒它们的线程的调度时间片（scheduling quantum）。结合双列表策略，时间片继承允许 Singularity 在一个 SIP 中的一个线程上的用户代码切换到另一个 SIP 中的另一个线程上的用户代码，仅需少至 394 个 CPU 周期。

### 3.4 垃圾回收

垃圾回收（garbage collection）是大多数安全语言的基本组成部分，因为它防止了可能颠覆安全保证的内存释放错误。在 Singularity 中，内核和 SIP 对象空间都是垃圾回收的。

大量现有的垃圾回收算法和经验强烈暗示，没有一种垃圾回收器适合所有系统或应用程序代码 [10]。Singularity 的架构解耦了每个 SIP 垃圾回收器的算法、数据结构和执行，使得回收器可以根据 SIP 中代码的行为来选择，并且无需全局协调即可运行。使这成为可能的 Singularity 四个方面是：每个 SIP 是一个封闭环境，拥有自己的运行时支持；指针不跨越 SIP 或内核边界，因此回收器无需考虑跨空间指针；通道上的消息不是对象，因此仅对消息和交换堆中的其他数据需要内存布局上的一致，而交换堆是引用计数的；内核控制内存页分配，这为协调资源分配提供了枢纽。

Singularity 的运行时系统目前支持五种回收器——分代半空间（generational semi-space）、分代滑动压缩（generational sliding compacting）、前两者的自适应组合、标记 - 清除（mark-sweep）和并发标记 - 清除（concurrent mark-sweep）。我们目前为系统代码使用并发标记 - 清除回收器，因为它在回收期间具有非常短的暂停时间。使用此回收器，每个线程拥有一个隔离的空闲列表（segregated free list），这在正常情况下消除了线程同步。垃圾回收在分配阈值时触发，并在一个独立的回收线程中执行，该线程标记可达对象。在回收期间，回收器停止每个线程以扫描其栈，这对于典型的栈引入了不到 100 微秒的暂停时间。此回收器的开销高于非并发回收器，因此我们通常为应用程序使用更简单的非并发标记 - 清除回收器。

每个 SIP 拥有自己的回收器，该回收器唯一负责回收其对象空间中的对象。从回收器的角度来看，进入或离开 SIP（或内核）的控制线程看起来类似于传统垃圾回收环境中来自本机代码的调用或回调。因此，不同对象空间的垃圾回收可以完全独立地调度和运行。如果一个 SIP 采用暂停世界（stop-the-world）回收器，即使线程因内核调用而在内核对象空间中运行，该线程也被视为相对于 SIP 对象空间已停止。然而，当线程返回 SIP 时，它将在回收期间被停止。

在垃圾回收环境中，线程栈包含作为回收器潜在根的对象引用。内核调用也在用户线程的栈上执行，并可能在此栈中存储内核指针。乍一看，这似乎通过创建跨 SIP 指针而违反了内存独立性不变量，并且至少使用户和内核的垃圾回收纠缠在一起。

为了避免这些问题，Singularity 界定了每个空间栈帧之间的边界，使得垃圾回收器无需看到另一个空间的引用。在跨空间（SIP → 内核或内核 → SIP）调用时，Singularity 将被调用者保存的寄存器保存到栈上的一个特殊结构中，该结构同时标记跨空间调用。这些结构标记了属于每个对象空间的栈区域边界。由于内核 ABI 中的调用不传递对象指针，垃圾回收器可以跳过来自另一个空间的帧。这些分隔符还有助于干净地终止 SIP。当一个 SIP 被杀死时，其每个线程立即被内核异常停止，该异常跳过并释放进程的栈帧。

### 3.5 通道实现

通道端点和跨通道传输的值驻留在交换堆中。端点不能驻留在 SIP 的对象空间中，因为它们可能被通过通道传递。类似地，在通道上传递的数据不能驻留在进程中，因为传递它将违反内存独立性不变量。消息发送者通过将指向消息的指针存储到接收端点中由消息交换协议当前状态确定的位置来转移所有权。然后，如果接收线程被阻塞等待接收消息，发送者通知调度器。

为了在通道上实现零分配（zero-allocation）通信，我们对每个通道的队列大小施加了一个有限性属性（finiteness property）。我们采纳的规则（由 Sing# 强制执行）是：契约状态转换中的每个环都必须包含至少一个接收和一个发送动作。这个简单规则保证了任何一方都不能在无需等待消息的情况下发送无界量的数据。它允许在端点中静态分配队列缓冲区。虽然这个规则看似限制性强，但我们在实践中尚未遇到需要放宽此规则的情况。

预分配端点队列并传递指向交换堆内存的指针，自然允许多 SIP 子系统（如 I/O 栈）的 " 零拷贝 " 实现。例如，磁盘缓冲区和网络数据包可以跨多个通道、通过协议栈传输到应用程序 SIP 中，而无需复制。

### 3.6 安全主体与访问控制

在 Singularity 中，应用程序是安全主体（security principal）。更准确地说，主体是复合的（compound），如同 Lampson 等人 [16, 29] 的定义：它们反映了当前 SIP 的应用程序身份、应用程序运行时的可选角色，以及通过其调用应用程序或授予其委托权限的可选主体链。传统意义上的用户是应用程序的角色（例如，以已登录用户角色运行的系统登录程序）。应用程序名称来源于 MBP 清单，清单又携带应用程序发布者的名称和签名。

在正常情况下，SIP 恰好与一个安全主体关联。然而，未来可能有必要允许每个 SIP 关联多个主体，以避免在某些行使多个不同主体权限的应用中过度创建 SIP。为了支持这种使用模式，我们允许通过通道将权限委托给现有 SIP。

SIP 之间的所有通信都通过通道进行。从保护资源（例如文件）的 SIP 角度来看，每个入站通道代表一个安全主体发言，该主体作为关于该通道的访问控制决策的主语。Singularity 的访问控制是自主的（discretionary）；例如，由文件系统 SIP 负责对其提供的对象执行控制。追踪主体名称并将主体与 SIP 和通道关联所需的内核支持相当少。访问控制决策通过将文本主体与模式（如正则表达式）匹配来做出，但此功能完全通过 SIP 空间的库提供，如 Wobber 等人 [30] 所讨论的。

---

## 4. 设计空间探索

在前几节中，我们描述了 Singularity 的架构基础：SIP、通道、MBP 和 Singularity 内核。在此基础之上，我们正在探索操作系统设计空间中的新领域。Singularity 的原则化架构、对健全静态分析的适宜性以及相对较小的代码库使其成为探索新方案的出色载体。在以下小节中，我们描述了 Singularity 中的四个探索方向：用于生成式编程的编译时反射、用于增强 SIP 的硬件保护域支持、硬件无关的异构多处理，以及类型化汇编语言。

### 4.1 编译时反射

Java 和 CLR 运行时环境提供了动态检查现有代码和元数据（类型和成员）以及在运行时生成新代码和元数据的机制。这些能力——通常称为反射（reflection）——使得高度动态的应用程序和生成式编程成为可能。

不幸的是，运行时反射的强大能力带来了高昂的代价。运行时反射需要大量的运行时元数据，它因未来可能的代码变更而抑制了代码优化，它可以被用来绕过系统安全和信息隐藏，且由于反射 API 的低层次特性而使代码生成变得复杂。

我们开发了一种新的编译时反射（Compile-Time Reflection, CTR）功能 [9]。编译时反射是 CLR 完整反射能力的部分替代。CTR 的核心特性是 Sing# 中一个名为 transform（变换）的高层次构造，它允许程序员以模式匹配和模板风格编写检查和生成代码。生成的代码和元数据可以被静态验证以确保其格式良好、类型安全，且不违反系统安全属性。同时，程序员可以避免反射 API 的复杂性。

变换可以由应用程序开发者提供，也可以由操作系统作为其可信计算基（trusted computing base）的一部分提供。变换可以在前端编译器中应用（当源代码转换为 MSIL 时），也可以在安装时应用（当 MSIL 被读取并在编译为本机指令之前）。在 SIP 的类型安全世界中，操作系统提供的变换可以强制执行系统策略并改善系统性能，因为它们可以安全地将可信代码引入到原本不可信的进程中。

#### 4.1.1 基于 CTR 的清单配置

CTR 的一个早期用途是从 MBP 清单中配置参数的声明式规约构建 SIP 启动的样板代码。生成的代码通过内核 ABI 检索启动参数，将参数转换为其声明的适当类型，并为 SIP 填充启动对象。此 CTR 变换完全替代了程序中传统的基于字符串的命令行参数处理，改用声明式的基于清单的配置。类似的变换用于从清单信息自动配置设备驱动程序 [25]。

清单 3 中的变换名为 `DriverTransform`，它从驱动程序资源需求的声明式声明生成设备驱动程序的启动代码。例如，SB16 音频驱动程序中的声明描述了通过 `IoPortRange` 类访问 I/O 寄存器组的需求，见清单 4。

`DriverTransform` 匹配该类，因为它派生自 `DriverCategoryDeclaration` 并包含指定的元素，例如适当类型的 `Values` 字段和私有构造函数的占位符。关键字 `reflective` 表示一个占位符，其定义将由使用 `implement` 修饰符的变换生成。占位符是前向引用，使程序中的代码能够引用随后由变换产生的代码。

变换中以 `$` 开头的是模式变量。在示例中，`$DriverConfig` 绑定到 `Sb16Config`。匹配多个元素的变量以两个 `$` 开头。例如，`$$ioranges` 表示一个字段列表，每个字段的类型 `$IoRangeType` 派生自 `IoRange`（各字段的类型不必相同）。为了为集合中的每个元素生成代码（如字段集合 `$$ioranges`），模板可以包含 `forall` 关键字，它为集合中的每个绑定复制模板。上述变换产生的结果代码等价于清单 5。

该示例还说明了由变换生成的代码可以在编译变换时进行类型检查，而不是像宏那样将此错误检查推迟到变换被应用时。在示例中，对 `Values` 的赋值是可验证安全的，因为构造对象的类型（`$DriverConfig`）与 `Values` 字段的类型匹配。

CTR 变换已被证明是生成式编程的有效工具。随着我们在 Singularity 的新领域中应用 CTR，我们继续改进变换的通用性。例如，最近用 CTR 变换生成编组（marshaling）代码的实验需要增加变换的表达能力。

### 4.2 硬件保护域

大多数操作系统使用 CPU 内存管理单元（MMU）硬件通过两种机制隔离进程。第一，进程仅被允许访问特定的物理内存页。第二，特权级别阻止不可信代码执行操作系统资源（如 MMU 或中断控制器）的特权指令。这些机制具有非平凡的性能开销，这些开销在很大程度上是隐性的，因为之前没有广泛使用的替代方案可供比较。

为了探索硬件保护与软件隔离之间的设计权衡，Singularity 最近的工作用保护域（protection domain）[1] 来增强 SIP。保护域可以在 SIP 周围提供额外一层基于硬件的保护。SIP 较低的运行时开销使其在比传统进程更细的粒度上使用成为可能，但硬件隔离作为软件隔离机制潜在故障的纵深防御（defense-in-depth），或在 32 位机器上提供所需的地址空间方面仍然有价值。

保护域是硬件强制的保护边界，可以承载一个或多个 SIP。每个保护域由一个独立的虚拟地址空间组成。处理器的 MMU 以传统方式强制内存隔离。每个域有自己的交换堆，用于域内 SIP 之间的通信。不将其 SIP 与内核隔离的保护域称为内核域（kernel domain）。内核域中的所有 SIP 运行在处理器的管理员特权级别（x86 的 ring 0），并共享内核的交换堆，从而简化了进程和内核之间的转换和通信。非内核域运行在用户特权级别（x86 的 ring 3）。

保护域内的通信继续使用 Singularity 高效的引用传递方案。然而，由于每个保护域位于独立的地址空间中，跨域通信需要数据复制或写时复制（copy-on-write）页映射。Singularity 通道的消息传递语义使这些实现对应用程序代码来说没有区别（性能除外）。

原则上，保护域可以承载包含不可验证代码（用 C++ 等不安全语言编写）的单个进程。虽然这对运行遗留代码非常有用，但我们尚未探索这种可能性。目前，保护域内的所有代码也包含在 SIP 中，SIP 继续提供隔离和故障遏制边界。

由于多个 SIP 可以承载在一个保护域内，域可以有选择地用于在特定进程之间或内核与进程之间提供硬件隔离。SIP 到保护域的映射是运行时决策。一个为每个 SIP 提供独立保护域的 Singularity 系统类似于完全硬件隔离的微内核系统，如 MINIX 3 [12]（见图 4a）。一个内核域承载内核、设备驱动程序和服务的 Singularity 系统类似于传统的单体操作系统，但对驱动程序或服务故障具有更强的弹性（见图 4b）。Singularity 还支持独特的硬件和软件隔离的混合组合，例如基于已签名代码选择内核域（见图 4c）。

#### 4.2.1 量化不安全代码税

Singularity 提供了一个独特的机会，可以在苹果对苹果的比较中量化硬件和软件隔离的成本。一旦了解了成本，各个系统就可以在硬件隔离的收益大于成本时选择使用硬件隔离。

硬件保护并非免费的，尽管其成本是分散的且难以量化。硬件保护的成本包括页表的维护、软 TLB 未命中、跨处理器的 TLB 维护、硬件分页异常、以及支持硬件保护的操作系统代码和数据造成的额外缓存压力。此外，TLB 访问处于许多处理器设计的关键路径上 [2, 15]，因此可能影响处理器的时钟速度和流水线深度。硬件保护增加了内核调用和进程上下文切换的成本 [3]。在具有无标签 TLB 的处理器上（如大多数当前的 x86 架构实现），进程上下文切换需要刷新 TLB，从而产生重填成本。

图 5 绘制了 WebFiles 基准测试在六种不同硬件和软件隔离配置下的归一化执行时间。WebFiles 是一个基于 SPECweb99 的 I/O 密集型基准测试。它由三个 SIP 组成：一个发出随机文件读取请求的客户端（文件大小服从 Zipf 分布）、一个文件系统和一个磁盘设备驱动程序。所有时间都相对于默认的 Singularity 配置进行了归一化，在该配置中三个 SIP 运行在与内核相同的地址空间和特权级别中，且分页硬件尽可能地被禁用。

WebFiles 基准测试清楚地展示了不安全代码税（unsafe code tax），即为不安全代码构建的系统中每个程序所付出的开销。打开 TLB 并使用 4KB 页的单一系统地址空间后，WebFiles 立即经历 6.3% 的减速。将客户端 SIP 移到独立的保护域（仍在 ring 0）后，减速增加到 18.9%。将客户端 SIP 移到 ring 3 后，减速增加到 33%。最后，将三个 SIP 各自移到独立的 ring 3 保护域后，减速增加到 37.7%。作为比较，安全代码的运行时开销低于 5%（通过禁用编译器中的数组边界和其他检查来测量）。

WebFiles 所经历的不安全代码税可能是最坏情况。并非所有应用程序都像 WebFiles 那样 IPC 密集，且很少有操作系统是完全隔离的、硬件保护的微内核。然而，几乎所有当今使用的系统都经历着在 ring 3 中运行用户进程的开销。在传统的硬件保护操作系统中，每个进程都支付不安全代码税，无论它包含的是安全代码还是不安全代码。SIP 提供了仅让不安全程序支付不安全代码税的选项。

### 4.3 异构多处理

由于物理限制，在芯片上复制处理器比提高处理器速度更容易。随着厂商已经展示了具有 80 个处理核心的原型芯片 [27]，我们已经开始在 Singularity 中进行所谓 " 众核 "（many-core）系统支持的实验。这些实验建立在 Singularity 内核已经提供的 SMP 支持之上。

操作系统对众核系统的支持不仅限于传统 SMP 系统在扩展时所需的简单线程安全和数据局部性问题。正如 Chakraborty 等人 [5] 的工作所暗示的，代码和元数据的局部性可以成为关键的性能瓶颈。Chakraborty 通过在用户 - 内核切换时动态切换处理器来提高系统性能，使操作系统代码运行在一组处理器上，应用程序代码运行在另一组上。他们断言这种处理器动态专业化实现了更好的指令缓存局部性，并且随着处理器为应用程序或操作系统代码特性进行自我调优，分支预测也得到了改善。我们预计这种动态专业化在每芯片核心数增长快于每芯片缓存时将变得更加有益。

Singularity 已经提供了超越 Chakraborty 在单体操作系统中所能实现的进一步处理器动态专业化机会。例如，由于许多传统操作系统服务——如文件系统和网络栈——位于各自的 SIP 中，Singularity 可以通过将处理器专用于特定 SIP 来实现众核处理器的专业化。每个处理器更小的代码足迹意味着 SIP 代码与 i-cache 及处理器中其他动态性能优化硬件之间应有更大的亲和性 [17]。

我们的假设是，更小的代码足迹将导致处理器更大的动态专业化。我们最近已经实验了在专用于特定 SIP 的处理器上仅运行 Singularity 微内核的精简子集。在最小变体中，专用 SIP 处理器上完全不运行内核。相反，所有 ABI 调用通过处理器间中断从专用处理器远程传送到运行完整内核的处理器。

#### 4.3.1 指令集架构

处理核心不仅在 CPU 中增殖，在 I/O 卡和外围设备中也在增殖。PC 中常见的可编程 I/O 卡包括图形处理器、网卡、RAID 控制器、声音处理器和物理引擎。这些处理核心提出了独特的挑战，因为它们通常与系统 CPU 具有非常不同的指令集架构和性能特征。

我们目前在操作系统社区中看到两种处理可编程 I/O 处理器的方法。一方是 " 传统主义者 "，他们认为可编程 I/O 处理器来来去去，因此长期来看没有必要在操作系统中考虑它们。可编程 I/O 设备应被视为 I/O 设备，其处理器作为实现细节隐藏在操作系统 I/O 抽象之后——例如，微软的 TCP Chimney 卸载架构 [20] 就遵循了这种方法。另一方是 " 专业主义者 "，他们认为 I/O 处理器（如 GPU）应被视为特殊的、独立的处理单元，在标准操作系统计算抽象之外执行——这是微软 DirectX 所遵循的方法。对于这一方，I/O 处理器将始终需要独特的工具集。

在 Singularity 中，我们看到了为可编程 I/O 处理器追寻新路线的机会。我们同意 " 专业主义者 " 的看法，即可编程 I/O 处理器会持续存在，因为专用处理器具有更好的性能功耗比（performance-per-watt）。然而，与 " 专业主义者 " 不同的是，我们正在探索一个假设：可编程 I/O 处理器应该成为操作系统调度和计算抽象的一等实体。核心思想非常简单：可编程 I/O 处理器上的计算应在 SIP 中进行。由于我们现有的异构处理支持，专用 I/O 处理器无需运行比将大多数 ABI 操作远程传送到 CPU 的最小变体更多的 Singularity 内核。

我们相信 Singularity 架构提供了五个优势，使这种面向可编程 I/O 处理的新设计前景看好。第一，SIP 最小化了对 I/O 核心上精细处理器特性的需求。例如，I/O 处理器无需拥有用于进程保护的内存管理单元。第二，基于契约的通道明确定义了 I/O 处理器上的 SIP 与其他 SIP 之间的通信。第三，Singularity 的内存隔离不变量消除了协处理器和 CPU 上 SIP 之间共享内存（或缓存一致性）的需求。第四，小型的、进程局部的 ABI 将可以安全地在本地实现的操作——如内存分配——与必须涉及其他 SIP 的服务隔离开来。最后，Singularity 以抽象的 MSIL 格式打包基于清单的程序，该格式可以转换为任何 I/O 处理器的指令集。同一个 TCP/IP 二进制文件可以为系统的 x86 CPU 和基于 ARM 的可编程网络适配器安装。

我们预计 Singularity MBP 以 MSIL 编码的指令集中立性最终甚至可能与众核 CPU 相关。随着众核系统的普及，工程界的许多人预期核心的硬件专业化。例如，大型乱序核心与小型顺序核心的配对将为系统提供对功耗的更大控制。众核系统使处理器专业化成为可能，因为每个单独的处理器无需支付单核芯片所要求的全部兼容性代价；一个众核芯片只要其中至少一个核心向后兼容即可被视为向后兼容。我们的假设是，Singularity 二进制文件可以像面向可编程 I/O 处理器上的 " 无遗留负担 " 核心那样容易地面向众核 CPU 上的 " 无遗留负担 " 核心。

### 4.4 类型化汇编语言

由于 Singularity 使用软件验证来强制 SIP 之间的隔离，其验证器的正确性对 Singularity 的安全至关重要。例如，为了确保 SIP 中的不可信代码不能访问 SIP 之外的内存，验证器必须检查代码不将任意整数强制转换为指针。目前，Singularity 依赖标准的 Microsoft Intermediate Language（MSIL）验证器来检查基本的类型安全属性（例如没有从整数到指针或从整数到内核句柄的强制转换）。Singularity 还有一个所有权检查器（ownership checker），验证 MSIL 代码遵守 Singularity 的规则：交换堆中的每个块仅由一个线程可访问。

Singularity 使用 Bartok 编译器 [13] 将 MBP 的 MSIL 代码翻译为本机机器语言代码（如 x86 代码）。如果编译器没有 bug，它将把安全的 MSIL 代码翻译为安全的本机代码。但由于 Bartok 是一个大型且高度优化的编译器，它可能包含 bug，其中一些 bug 可能导致编译器将安全的 MSIL 代码翻译为不安全的本机代码。

我们已经开始将携证代码（proof-carrying code）[22] 和类型化汇编语言（typed assembly language）[21] 的研究整合到 Bartok 和 Singularity 中。Bartok 有一个类型化中间语言，在编译到本机代码的过程中维护类型信息。这些信息将允许 Singularity 验证本机 " 类型化汇编语言 "（TAL）代码的安全性，而非 MSIL 代码。此外，本机代码验证器将允许 Singularity 运行由其他编译器生成或手写的安全本机代码。

当 Bartok 将 MSIL 代码编译为本机代码时，它将数据布局从 MSIL 的抽象数据格式翻译为具体数据格式。此具体格式精确指定了字段在对象中的位置、方法指针在方法表中的位置、以及运行时类型信息的位置。为方法表和运行时类型信息生成类型是具有挑战性的；使用简单的记录类型天真地处理这些问题可能导致类型不正确的本机代码（或更糟，一个不健全的类型系统；更多信息参见 [18]）。另一方面，使用过于复杂的类型系统可能使类型检查变得困难甚至不可判定。Bartok 使用 Chen 和 Tarditi 的 LIL_C 类型系统 [6]，该系统可以对方法表和运行时类型信息的布局进行类型化，同时仍然具有简单的类型检查算法。

具体数据格式还指定了垃圾回收信息，例如每个对象中指针字段的位置。如果此信息有误，垃圾回收器可能过早地回收数据，留下不安全的悬空指针。此外，Singularity 的某些垃圾回收器对 SIP 的本机代码施加了额外要求。例如，分代回收器要求代码在写入字段之前调用 " 写屏障 "（write barrier）。未能调用写屏障可能导致悬空指针。最后，每个 Singularity 垃圾回收器目前都是用不安全的 Sing# 代码编写的，此代码中的 bug 可能破坏 Singularity 的安全性。我们正在通过将垃圾回收器重写为安全代码来解决这些问题，以便我们可以同时验证 SIP 的不可信代码和回收器的代码。

在安全语言中编写垃圾回收器是具有挑战性的，因为传统的安全语言不支持显式内存释放。因此，我们开发了一个类型系统，既支持 LIL_C 的类型又支持关于内存状态的显式证明 [11]。使用此类型系统，垃圾回收器可以静态证明其释放操作是安全的。该类型系统对 LIL_C 和内存证明的双重支持将允许 SIP 的不可信代码和垃圾回收器代码共存为一个单一的、可验证的 TAL 程序，确保 SIP 代码、回收器及其交互的安全性。我们目前仅实现了一个简单的复制回收器（copying collector），用简单的 RISC 类型化汇编语言编写，但我们正在将更复杂的回收器移植到我们 TAL 的 x86 版本。

---

## 5. 结论

我们在三年多前启动了 Singularity 项目，以探索语言、工具和操作系统方面的进展如何产生更可靠的软件系统。我们的目标仍然是开发能够显著改善可靠性的技术和手段。作为共同的实验室，我们创建了 Singularity 操作系统。该操作系统的核心是三个基本的架构决策，它们显著提高了验证软件系统预期行为的能力：软件隔离进程（SIP）、基于契约的通道和基于清单的程序（MBP）。

### 5.1 性能与兼容性

Singularity 项目放弃了对大多数成功操作系统至关重要的两个优先事项：高性能和与先前系统的兼容性。Singularity 始终偏好设计清晰性（design clarity）而非性能。在大多数决策点上，Singularity 尝试提供 " 足够好 " 的性能，但不追求更好。在某些情况下，如通信微基准测试，Singularity 显著优于现有系统。这种性能是架构和设计决策的愉快副产品，其动机是构建更可靠的系统。

Singularity 放弃了应用程序和驱动程序的兼容性以探索新的设计选项。这一选择是一把双刃剑。一方面，我们可以自由地在没有遗留约束的情况下探索新想法。另一方面，我们被迫重写或移植 Singularity 系统中的每一行代码。我们不会建议每个项目都采用这种方法，但我们相信这对 Singularity 来说是正确的选择。研究自由带来的回报值得付出的代价。

### 5.2 架构、语言与工具的协同

我们从 Singularity 中学到的最重要教训之一是编程语言、操作系统架构和验证工具之间紧密反馈循环的益处。一个领域的进步可以产生有益的影响，进而推动另一个领域的进步；该进步又使另一个领域的进步成为可能，循环继续（见图 6）。

例如，我们将 SIP 密封以防止代码加载的决策受到了微软 Office 产品团队一位代表的启发，该代表对某个特定插件的低质量感到担忧。这一操作系统架构的改变之所以可以强制执行，是因为 SIP 包含安全代码——没有代码通过指针被修改——这增强了静态验证技术的健全性和覆盖范围。

类似地，Singularity 中通道和契约的设计受到了我们同事在 Web 服务通信验证方面最新进展的深刻影响。我们操作系统社区中的同事经常指出，在实现层面，两条单向通道优于一条双向通道。然而，双向通道使两个进程之间的通信更容易分析。遵循 Web 服务设计不仅改善了验证，还通过指针传递实现零拷贝通信而改善了消息传递性能。

语言和验证技术的选择影响操作系统架构。操作系统架构的选择影响语言和验证技术。Singularity 仅仅触及了将语言、工具和架构一起考虑所带来的好处的表面，但它凸显了可用的机遇。

---

## 6. 致谢

Mark Aiken、Manuel Fähndrich、Chris Hawblitzel 和 Ted Wobber 分别对第 4.2 节、第 4.1 节、第 4.4 节和第 3.6 节做出了特别贡献，在此致以特别感谢。

像 Singularity 这样的项目是众多人共同努力的成果。Singularity 项目受益于微软研究院许多杰出同事的才智，包括来自班加罗尔实验室的 Sriram Rajamani；来自剑桥实验室的 Paul Barham、Richard Black、Austin Donnelly、Tim Harris、Rebecca Isaacs 和 Dushyanth Narayanan；来自雷德蒙德实验室的 Mark Aiken、Juan Chen、Trishul Chilimbi、John DeTreville、Manuel Fahndrich、Wolfgang Grieskamp、Chris Hawblitzel、Orion Hodson、Mike Jones、Steven Levi、Qunyan Mangus、Virendra Marathe、Kirk Olynyk、Mark Plesko、Jakob Rehof、Wolfram Schulte、Dan Simon、Bjarne Steensgaard、David Tarditi、Songtao Xia、Brian Zill 和 Ben Zorn；以及来自硅谷实验室的 Martin Abadi、Andrew Birrell、Úlfar Erlingsson、Roy Levin、Nick Murphy、Chandu Thekkath 和 Ted Wobber。迄今为止已有 22 名实习生为 Singularity 贡献了大量的代码和灵感。

最后，微软工程部门的许多个人也对 Singularity 做出了直接贡献——其中许多是在业余时间。

---

## 7. 参考文献

[1] Aiken, M., Fähndrich, M., Hawblitzel, C., Hunt, G. and Larus, J., Deconstructing Process Isolation. In *Proceedings of the ACM SIGPLAN Workshop on Memory Systems Correctness and Performance (MSPC 2006)*, San Jose, CA, October 2006.

[2] Allen, D.H., Dhong, S.H., Hofstee, H.P., Leenstra, J., Nowka, K.J., Stasiak, D.L. and Wendel, D.F. Custom Circuit Design as a Driver of Microprocessor Performance. *IBM Journal of Research and Development*, 44 (6) 2000.

[3] Anderson, T.E., Levy, H.M., Bershad, B.N. and Lazowska, E.D. The Interaction of Architecture and Operating System Design. In *Proceedings of the Fourth International Conference on Architectural Support for Programming Languages and Operating Systems*, pp. 108-120, Santa Clara, CA, 1991.

[4] Bershad, B.N., Savage, S., Pardyak, P., Sirer, E.G., Fiuczynski, M., Becker, D., Eggers, S. and Chambers, C. Extensibility, Safety and Performance in the SPIN Operating System. In *Proceedings of the Fifteenth ACM Symposium on Operating System Principles*, pp. 267-284, Copper Mountain Resort, CO, 1995.

[5] Chakraborty, K., Wells, P. and Sohi, G., Computation Spreading: Employing Hardware Migration to Specialize CMP Cores On-the-fly. In *12th International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS XII)*, pp. 283-302, San Jose, CA, October 2006.

[6] Chen, J. and Tarditi, D., A Simple Typed Intermediate Language for Object-oriented Languages. In *Proceedings of the 32nd ACM SIGPLAN-SIGACT Symposium on Principles of Programming Languages (POPL '05)*, pp. 38-49, Long Beach, CA, January 2005.

[7] ECMA International, ECMA-335 Common Language Infrastructure (CLI), 4th Edition. Technical Report 2006.

[8] Fähndrich, M., Aiken, M., Hawblitzel, C., Hodson, O., Hunt, G., Larus, J.R. and Levi, S., Language Support for Fast and Reliable Message Based Communication in Singularity OS. In *Proceedings of the EuroSys 2006 Conference*, pp. 177-190, Leuven, Belgium, April 2006.

[9] Fähndrich, M., Carbin, M. and Larus, J., Reflective Program Generation with Patterns. In *5th International Conference on Generative Programming and Component Engineering (GPCE'06)*, Portland, OR, October 2006.

[10] Fitzgerald, R. and Tarditi, D. The Case for Profile-directed Selection of Garbage Collectors. In *Proceedings of the 2nd International Symposium on Memory Management (ISMM '00)*, pp. 111-120, Minneapolis, MN, 2000.

[11] Hawblitzel, C., Huang, H., Wittie, L. and Chen, J., A Garbage-Collecting Typed Assembly Language. In *ACM SIGPLAN Workshop on Types in Language Design and Implementation (TLDI '07)*, Nice, France, January 2007.

[12] Herder, J.N., Bos, H., Gras, B., Homburg, P. and Tanenbaum, A.S. MINIX 3: A Highly Reliable, Self-Repairing Operating System. *Operating System Review*, 40 (3). pp. 80-89. July 2006.

[13] Hunt, G., Larus, J., Abadi, M., Aiken, M., Barham, P., Fähndrich, M., Hawblitzel, C., Hodson, O., Levi, S., Murphy, N., Steensgaard, B., Tarditi, D., Wobber, T. and Zill, B., An Overview of the Singularity Project. Technical Report MSR-TR-2005-135, Microsoft Research, 2005.

[14] Hunt, G., Larus, J., Abadi, M., Aiken, M., Barham, P., Fähndrich, M., Hawblitzel, C., Hodson, O., Levi, S., Murphy, N., Steensgaard, B., Tarditi, D., Wobber, T. and Zill, B., Sealing OS Processes to Improve Dependability and Safety. In *Proceedings of the EuroSys 2007 Conference*, pp. 341-354, Lisbon, Portugal, March 2007.

[15] Kongetira, P., Aingaran, K. and Olukotun, K. Niagara: A 32-Way Multithreaded Sparc Processor. *IEEE Micro*, 25 (2). pp. 21-29. March 2005.

[16] Lampson, B., Abadi, M., Burrows, M. and Wobber, E.P. Authentication in distributed systems: Theory and Practice. *ACM Transactions on Computer Systems*, 10 (4). pp. 265-310. November 1992.

[17] Larus, J.R. and Parkes, M. Using Cohort-Scheduling to Enhance Server Performance. In *Proceedings of the USENIX 2002 Annual Conference*, pp. 103-114, Monterey, CA, 2002.

[18] League, C. A Type-Preserving Compiler Infrastructure, Yale University, New Haven, CT, December, 2002.

[19] Levy, H.M. Capability-Based Computer Systems. Butterworth-Heinemann, Newton, MA, 1984.

[20] Microsoft Corporation, Scalable Networking: Network Protocol Offload - Introducing TCP Chimney. Technical Report 2004.

[21] Morrisett, G., Walker, D., Crary, K. and Glew, N. From System F to Typed Assembly Language. *ACM Transactions on Programming Languages and Systems*, 21 (3). pp. 527-568. May 1999.

[22] Necula, G.C. and Lee, P., Safe Kernel Extensions Without Run-Time Checking. In *Proceedings of the Second Symposium on Operating System Design and Implementation*, Seattle, WA, October 1996.

[23] Saltzer, J.H. and Schroeder, M.D. The protection of information in computer systems. *Proceedings of the IEEE*, 63 (9). pp. 1268-1308. September 1975.

[24] Shapiro, J.S., Smith, J.M. and Farber, D.J. EROS: a Fast Capability System. In *Proceedings of the 17th ACM Symposium on Operating Systems Principles (SOSP '99)*, pp. 170-185, Charleston, SC, 1999.

[25] Spear, M.F., Roeder, T., Hodson, O., Hunt, G.C. and Levi, S., Solving the Starting Problem: Device Drivers as Self-Describing Artifacts. In *Proceedings of the EuroSys 2006 Conference*, pp. 45-58, Leuven, Belgium, April 2006.

[26] Swinehart, D.C., Zellweger, P.T., Beach, R.J. and Hagmann, R.B. A Structural View of the Cedar Programming Environment. *ACM Transactions on Programming Languages and Systems*, 8 (4). pp. 419-490. October 1986.

[27] Vangal, S., Howard, J., Ruhl, G., Dighe, S., Wilson, H., Tschanz, J., Finan, D., Iyer, P., Singh, A., Jacob, T., Jain, S., Venkataraman, S., Hoskote, Y. and Borkar, N., An 80-Tile 1.28TFLOPS Network-on-Chip in 65nm CMOS. In *2007 IEEE International Solid-State Circuits Conference*, San Francisco, CA, February 2007.

[28] von Behren, R., Condit, J., Zhou, F., Necula, G.C. and Brewer, E. Capriccio: Scalable Threads for Internet Services. In *Proceedings of the Nineteenth ACM Symposium on Operating Systems Principles (SOSP '03)*, pp. 268-281, Bolton Landing, NY, 2003.

[29] Wobber, E.P., Abadi, M., Burrows, M. and Lampson, B. Authentication in the Taos Operating System. *ACM Transactions on Computer Systems*, 12 (1). pp. 3-32. February 1994.

[30] Wobber, T., Abadi, M., Birrell, A., Simon, D.R. and Yumerefendi, A., Authorizing Applications in Singularity. In *Proceedings of the EuroSys 2007 Conference*, pp. 355-368, Lisbon, Portugal, March 2007.

<!-- obsidian-publish-source: Operating System 操作系统/Pappers 论文/SOSP/Singularity 重新审视软件栈.md -->