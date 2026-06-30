1. [Lua in the kernel? \[LWN.net\]](https://lwn.net/Articles/830154/)
2. [GitHub - luainkernel/lunatik: Lunatik is a framework for scripting the Linux kernel with Lua. · GitHub](https://github.com/luainkernel/lunatik)
3. [netbsd.org/\~lneto/dls14.pdf](https://netbsd.org/~lneto/dls14.pdf)

**作者：** Lourival Vieira Neto（The NetBSD Foundation）、Roberto Ierusalimschy（PUC-Rio 信息学系）、Ana Lúcia de Moura（PUC-Rio 信息学系）、Marc Balmer（The NetBSD Foundation）

![[Pasted image 20260630192841.png]]
## 摘要

Extensible operating system（可扩展操作系统）是一种基于以下理念的设计范式：操作系统可以通过允许用户扩展来适应用户需求。在另一种场景——应用程序开发——中，存在一种范式主张复杂系统应当允许用户编写脚本，以便将应用程序定制为满足其需求的形态。本文提出了 scriptable operating system（可脚本化操作系统）的概念，将脚本化开发范式应用于可扩展操作系统的理念之中。

可脚本化操作系统主张：操作系统可以通过允许用户对其内核进行脚本化编程来充分提供可扩展性。本文还展示了一种 kernel-scripting environment（内核脚本化环境）的实现，该环境允许用户使用脚本语言 Lua 对 Linux 和 NetBSD 操作系统进行动态扩展。为评估该环境，我们对两种操作系统内核进行了扩展，使用户能够使用 Lua 对 CPU frequency scaling（CPU 频率调节）和 network packet filtering（网络数据包过滤）进行脚本化编程。

**分类与主题描述符：** D.3.3 [语言构造与特性]：框架；D.4.7 [组织与设计]

**关键词：** scriptable operating system（可脚本化操作系统），kernel scripting（内核脚本化），Lua programming language（Lua 编程语言）

---

## 1. 引言

Extensible operating systems（可扩展操作系统）于 20 世纪 60 年代末被提出，作为一种设计方法，其核心主张是操作系统（OS）可以通过允许使用扩展来提高其灵活性。其基本思想在于，通用操作系统无法预见其所有应用程序的需求，因此应当能够调整自身行为以满足特定的或新兴的需求。

操作系统的可扩展性可以通过多种方式提供，从调整系统参数 —— 如 sysctl [23] 和 sysfs [24] 所支持的方式—— 到向内核动态注入或链接代码。后一种方法允许用户直接访问操作系统内部机制，并创建新的策略与机制；在过去二十年中，该方法在 Exokernel [8]、SPIN [6]、VINO [34]、µChoices [7] 和 Singularity [13] 等研究型操作系统中得到了广泛探索。目前大多数通用操作系统通过允许特权用户动态加载 kernel modules（内核模块）来提供这种可扩展性。

在另一种场景——可定制应用程序的开发——中，存在一种重要趋势，即将复杂系统分为两个部分——core（核心）和 configuration（配置）——并结合使用 system programming languages（系统编程语言）和 scripting languages（脚本语言）来开发这两个部分 [15, 26]。系统编程语言，如 C 和 C++，通常是编译型且静态类型的，用于开发应用程序的核心组件。脚本语言，如 Lua、Tcl 和 Python，通常是解释型且动态类型的。在此场景中，脚本语言用于实现配置部分，该部分负责连接应用程序的核心组件，将应用程序定制为满足用户个性化需求的形态。使用脚本语言来支持应用程序的定制化带来了显著的优势。首先，它提高了生产力，有利于构建敏捷的开发环境 [26]。其次，使用一种功能完备的语言允许用户开发运行时配置过程，而这些过程仅通过选择参数值是无法实现的 [15]。

在本工作中，我们展示了一项旨在提高操作系统灵活性的实验，该实验将可扩展操作系统与扩展脚本语言相结合。为探索我们所称的 scriptable operating systems（可脚本化操作系统）这一综合方法，我们首先开发了 Lunatik，这是一个小型子系统，它基于 Lua 编程语言 [14, 15] 为操作系统内核脚本化提供编程与执行环境。随后，我们将这一初始实验发展为一个更加可靠的基础设施——Lua in the NetBSD kernel（NetBSD 内核中的 Lua），该基础设施现已成为 NetBSD 官方发行版的一部分。这两种内核脚本化实现并未提供一个完全可脚本化的操作系统，而是提供了有助于使内核子系统可脚本化的框架。

本文其余部分组织如下：第 2 节讨论我们关于可脚本化操作系统的概念及其涉及的若干问题；第 3 节阐述选择 Lua 作为操作系统内核脚本语言的动机；第 4 节描述我们基于 Lua 的内核脚本化环境；第 5 节通过展示我们为 Linux 和 NetBSD 内核子系统实现的扩展来评估该环境；第 6 节讨论一些相关工作；最后，第 7 节给出本文的结论。

---

## 2. 可脚本化操作系统

操作系统既可以在其 user space（用户空间）部分（即系统程序）中实现可脚本化，也可以在其内核中实现可脚本化。事实上，大多数操作系统已经拥有可脚本化的系统程序，例如 BSD Init，它使用一组 shell scripts（Shell 脚本）来配置用户级初始化 [23]。理想情况下，一个可脚本化操作系统应当同时支持用户空间和内核的脚本化设施，提供一个通用的脚本语言环境来定制和配置整个系统。然而，我们关于可脚本化操作系统概念的核心与根本思想是 kernel scripting（内核脚本化）：可脚本化操作系统必须通过允许用户将脚本动态加载到内核中来提供可扩展性。由于这也是使操作系统可脚本化过程中最具挑战性且最为重要的部分，它是本文的研究重点。

操作系统内核脚本化可以通过两种不同的方式提供：embedding（嵌入）和 extending（扩展）一种脚本语言 [21, 25]。嵌入一种脚本语言意味着内核子系统充当宿主程序，将脚本解释器作为库调用以执行用户定义的脚本。扩展一种脚本语言意味着内核子系统充当用户定义脚本的库，由脚本掌握执行控制流。扩展一种语言将内核视为库：控制流从脚本开始，流入内核；嵌入一种语言将内核视为框架：控制流从内核开始，流入脚本。

为了通过嵌入脚本语言来提供可扩展性，内核开发者需要修改其子系统，使其通过调用脚本解释器来执行用户扩展脚本。通过这些脚本，用户可以根据自身需求调整操作系统的行为，定义适当的策略与机制。

通过嵌入脚本语言进行内核扩展的一些可能用途包括：

- **Process scheduling（进程调度）：** 通过对内核进行脚本化编程，用户可以定义替代性的进程和/或线程调度算法与策略。例如，该功能可以支持进程 QoS（Quality of Service，服务质量）的实现。这类扩展已被 Bossa 系统 [4] 成功探索，该系统使用了一种适用于此场景的 DSL（Domain-Specific Language，领域特定语言）。

- **Packet filtering（数据包过滤）：** 内核脚本化允许使用更复杂的规则来过滤和处理网络数据包，作为简单规则表的替代方案。某些可扩展操作系统（如 Exokernel [8]）通过领域特定语言提供了此类功能。

- **Device drivers（设备驱动程序）：** 内核开发者（及用户）可以使用脚本实现和定制设备驱动程序。例如，开发者可以为新的 USB（Universal Serial Bus，通用串行总线）外设编写驱动程序，或创建自定义的键盘布局。

将脚本语言扩展以支持内核脚本编程，需要开发 binding libraries（绑定库）来向用户脚本暴露内核函数和数据结构。为加载到内核空间的脚本提供对内核内部组件的直接访问，可以避免使用传统的用户级接口——后者通常涉及连续的高开销上下文切换和缓冲区复制。

通过扩展脚本语言实现内核脚本编程的一些可能场景包括：

- **Web servers（Web 服务器）：** Web 服务器可以通过在内核中加载脚本来处理 HTTP/SPDY [5, 9] 协议栈，从而提升性能。该用例已在许多可扩展操作系统中得到探索 [3, 19, 32, 35]。

- **File systems（文件系统）：** 内核脚本允许用户定义自己的磁盘和网络数据存储抽象。例如，用户可以定义自己的磁盘块布局，或为远程文件存储设施实现自定义协议。Exokernel [8] 通过使用 DSL 提供了用户自定义磁盘块布局的功能。

- **Network protocols（网络协议）：** 内核脚本可用于实现网络协议守护进程。此功能对于驻留在较低网络层的协议尤为有用。例如，LLDP（Link Layer Discovery Protocol，链路层发现协议）[2] 的用户级守护进程需要实现用户级原始套接字。应用层的 RTP（Real-time Transport Protocol，实时传输协议）[31] 与传输层协议有诸多共同之处，也是一个值得关注的例子。

除了提高操作系统的灵活性之外，内核脚本编程还为新内核算法和机制的原型开发及快速便捷的实验提供了一个有价值的环境。这种环境使内核开发者自身也能够采用快速开发方法学 [26]。

传统操作系统使用多种领域特定语言进行系统配置。例如，NetBSD 使用 DSL 进行系统初始化（rc.conf(5)）、数据包过滤（npf.conf(8)）以及内核静态配置（autoconf(9)）。可脚本化操作系统的方法用单一的脚本语言引擎替代这些不同的 DSL，从而简化了系统架构。

### 2.1 内核脚本编程示例

Linux 内核子系统 CPUfreq [27] 利用现代处理器提供的 dynamic frequency scaling（动态频率调节）机制来管理 CPU 频率和电压。一组内置策略根据特定需求控制系统的功耗。实现活跃策略的例程被周期性调用，以调整 CPU 频率来满足其关联的需求。

CPUfreq 的内置策略 Ondemand 以节能同时最小化性能损失为目标来控制 CPU 频率。该控制基于一段时间内的 CPU 负载：当 CPU 负载上升超过 " 上 " 阈值时，控制器将频率调至最大可用值；当 CPU 负载降至 " 下 " 阈值以下时，控制器将频率调至不超过当前频率百分之二十的最高可用值。

内核脚本编程可以为内置策略未能满足的需求提供支持。例如，用户定义的脚本可以控制 CPU 频率以防止过热，同时尽力保持系统性能并节省功耗。为实现该新策略，脚本可以扩展 Ondemand 算法，在分析 CPU 利用率之前先考虑 CPU 温度：如果当前温度高于或等于一百摄氏度，策略例程将忽略性能评估并降低频率以冷却处理器。图 1 展示了一段实现该 CPU 频率调节策略的 Lua 脚本。在该脚本中，函数 `cpufreq.target` 负责更改 CPU 频率。它接收 CPU 标识号、频率目标值以及一个参数——指定所设频率应为大于或等于目标值的最低频率（`'>='`）还是小于或等于目标值的最高频率（`'<='`）。

```lua
up = 80
down = 30
overheated = 100

function throttle(cpu, cur, max, min)
  -- 获取自上次检查以来的利用率
  local load = get_load(cpu)
  -- 获取温度
  local temp = acpi.get_temp(cpu)
  if temp >= overheated then
    -- 将频率降低 20%
    cpufreq.target(cpu, cur * 80 / 100, '<=')
  else
    if load > up then
      -- 将频率提升至最大值
      cpufreq.target(cpu, max, '>=')
    elseif load < down then
      -- 将频率降低 20%
      cpufreq.target(cpu, cur * 80 / 100, '<=')
    end
  end
end
```
**图 1.** 用于控制 CPU 频率的 Lua 脚本

内核脚本编程还可以提供和扩展网络数据包处理功能。NetBSD 内核子系统 NPF [29] 允许用户使用领域特定语言定义数据包过滤规则。这些用户定义的规则应用于网络流量，但仅限于第三层和第四层。为了执行 deep packet inspection（深度包检测）——即检查第四层以上的网络层——内核脚本环境可以允许用户将扩展脚本与常规 NPF 规则相关联。

假设在某个 SSH（Secure Shell，安全外壳协议）[36] 的特定实现中发现了一个新的漏洞，而该实现目前被防火墙后的一些服务器所采用。为避免这些服务器遭到入侵，我们可以使用一段 Lua 脚本来检查出站流量，并过滤来自运行该漏洞 SSH 实现的服务器的 SSH 数据包。阻断来自这些服务器的流量可以保护它们免受利用其漏洞的访问。

图 2 展示了实现该过滤功能的 Lua 扩展脚本。为激活过滤功能，我们可以将该脚本关联到一条 NPF 规则，使其应用于 TCP 端口 22 上的出站数据包。

函数 `filter` 接收网络数据包的头部（hdr）和载荷（pld）。由于它仅接收源自端口 22 的 TCP 数据包，因此可以假定载荷包含 SSH 消息。在 SSH 协议中，当连接建立后，双方都会发送一个标识字符串；该脚本解析此字符串以验证消息是否由运行存在漏洞的 SSH 实现的服务器发送。若是，则发出信号指示应丢弃该数据包。

为提取 SSH 实现版本，函数 `filter` 将载荷的前 255 字节转换为 Lua 字符串，并使用模式匹配来定位和提取 SSH 版本。该脚本使用的模式匹配以 "SSH-" 开头的字符串（在 Lua 模式中 `'%'` 字符用作转义符），后跟一个或多个除连字符和空白符以外的可打印字符，然后是一个连字符，再跟一个或多个除连字符和空白符以外的可打印字符（即指定版本号的部分）。

```lua
function filter(hdr, pld)
  -- 获取载荷的一个片段
  local seg = pld:segment(0, 255)
  -- 将片段数据转换为字符串
  local str = tostring(seg)
  -- 用于捕获软件版本的模式
  local pattern = 'SSH%-[^-%G]+%-([^-%G]+)'
  -- 获取软件版本
  local software_version = str:match(pattern)
  if software_version == 'OpenSSH_6.4' then
    -- 拒绝该数据包
    return false
  end
  -- 接受该数据包
  return true
end
```
**图 2.** 用于检查 SSH 数据包的 Lua 脚本

### 2.2 解决可脚本化操作系统的相关问题

由于可脚本化操作系统允许用户在特权（内核）模式下加载和运行代码，因此它们面临与先前可扩展操作系统研究 [30, 33] 相同类别的问题。又由于采用的是典型的脚本开发实践，它们也涉及常规可脚本化应用场景中存在的问题。

可脚本化操作系统的问题主要与维护系统完整性、提供开发便利性以及确保内核脚本的有效性和效率相关。在这些问题中，为操作系统内核提供脚本功能时的首要关切是保持其完整性。内核脚本不应被允许造成任何损害。换言之，不应允许脚本——无论有意还是无意——向系统本身或运行在其上的应用程序引入故障。在实践中，脚本可以通过多种方式损害系统完整性，例如：

- **Correctness（正确性）：** 内核脚本可能向系统引入错误行为，如导致系统崩溃或数据损坏。
- **Isolation（隔离性）：** 由特定用户加载的内核脚本可能破坏其他用户拥有的资源，或损害系统的公平性。
- **Liveliness（活性）：** 内核脚本可能陷入无限循环、阻塞执行流，或运行时间过长以至于损害整个系统的响应性。

传统操作系统通常试图通过仅允许特权用户加载和运行内核扩展来保证系统完整性。另一方面，可扩展操作系统通常允许用户在其内核中加载和运行非特权代码。可脚本化操作系统可以兼用两种方法，为内核嵌入式解释器的不同实例提供不同的特权级别。然而，由于脚本环境的高级特性，无论是特权还是非特权的内核脚本，都不能完全承担保证系统完整性的责任。特别是，即使是特权内核脚本也不应负责管理内存分配或显式同步。该责任应限制在系统编程语言代码中，以保持脚本环境中典型的角色分离。也就是说，我们应当防止内核脚本因管理内存分配（例如空指针解引用、内存泄漏）和显式同步（例如死锁、饥饿）的问题而损害系统完整性。系统语言应用于实现核心和底层操作（如内存分配和同步），而脚本语言应用于实现高层和配置部分（如资源分配策略）。

可扩展操作系统经常使用编程语言层面的资源来防止扩展故障并保证系统完整性。例如，Exokernel 提供了一组领域特定的扩展语言，允许用户创建自定义的磁盘和网络抽象；这些语言具有严格的限制，以防止扩展对系统造成损害（它们不是 Turing-complete languages（图灵完备语言））。Singularity [13] 提供了静态代码分析工具，同样用于防止扩展违规。

脚本语言通常提供保护特性，如 type safety（类型安全）、automatic memory management（自动内存管理）和 protected calls（受保护调用）。某些脚本语言还支持使用 sandboxing（沙箱）技术来保护资源。例如，Lua 可以限制每个解释器实例可用的库集合、使用的内存量以及执行的指令数。这种沙箱机制也可用于保障整体系统完整性。

可脚本化操作系统的另一个关键特性是开发便利性。脚本语言本质上是非常高级的编程语言。通常，它们具有动态类型，并提供高级数据结构、操作和 API。某些脚本语言还具有可扩展性，允许针对特定应用领域进行定制。这种定制能力支持领域特定语言的实现，同时具有在不同领域之间共享通用语法和 API 的优势。脚本语言也常用于提高生产力，并面向非程序员群体。例如，Lua 被用作 Wikipedia、World of Warcraft 和 Wireshark 的扩展语言；在这些应用中，Lua 通常被非程序员或非系统程序员使用。

内核脚本的有效性（effectiveness）和效率（efficiency）对于可脚本化操作系统同样极为重要。操作系统内核对时序问题特别敏感，因此内核脚本必须具有合理的效率。它们不应引入可能损害系统整体性能的开销。扩展操作系统内核的主要论据之一是避免内核空间与用户空间之间连续上下文切换所带来的开销。旨在从内核空间运行中获益的脚本，其引入的开销不应高于上下文切换所带来的开销。某些现代脚本语言解释器运行速度相当快，并在基准测试中证明了其效率。然而，如果在特定场景中效率确实是一个关切，一种提高可脚本化系统效率的策略是使用系统编程语言实现性能关键任务，并为脚本语言提供适当的绑定。

除性能问题外，内核脚本的有效性同样重要。内核脚本必须能够实现有用的扩展。这一需求可以通过在脚本语言与内核之间创建适当的绑定来实现。为允许创建此类绑定，脚本语言必须能够被嵌入和被扩展。它必须可定制以支持适合内核的编程接口和数据结构；它还必须提供适当的接口，以允许内核执行流调用脚本过程和访问脚本数据结构。绑定在使脚本易于编程和保持系统完整性方面也起着重要作用，因为它们为暴露给内核脚本的 API 和数据结构提供了适当的抽象和保护层级。

创建适当的绑定是使操作系统内核可脚本化时最困难的任务之一。适当的绑定是那些适合解决本节所述问题的绑定，即满足维护系统完整性、提供开发便利性以及确保内核脚本有效性和效率需求的绑定。操作系统内核通常表现为一组复杂且相互依赖的大型程序，因此这一任务的困难性是这些系统复杂本质所固有的。然而，如果脚本语言提供了适当的嵌入和扩展资源，则可以减轻这种困难。

---

## 3. 为什么选择 Lua？

Lua 是一种可扩展的扩展编程语言（extensible extension programming language），其设计目标是既能定制应用程序，也能被应用程序所定制 [15, 17]。与大多数脚本语言不同，Lua 被专门设计为一种 embedded language（嵌入式语言）：它以标准 C 库的形式实现，并提供定义良好的 API [14, 18]。

Lua C API 提供了一种务实的方式，可在双向上将脚本绑定到宿主程序（host program）[18]。通过 Lua C API，宿主应用程序可以获取和设置脚本中定义的变量，并调用脚本中定义的函数——这一能力使 Lua 成为一种 extension language（扩展语言）。Lua C API 还允许宿主程序向脚本导出函数和变量，为 Lua 环境添加新的功能设施——这一能力使 Lua 成为一种 extensible language（可扩展语言）。

扩展操作系统与扩展用户级应用程序有所不同，因为内核受到一组特殊约束的制约。操作系统内核通常使用 C 编程语言的有限子集编写。由于 floating-point unit（浮点运算单元）的上下文切换开销过大，内核不支持浮点类型。同时，内核也不支持完整的 C 标准库；取而代之的是仅有少量标准函数可用。例如，内核代码无法使用传统的 free/malloc 函数，因为这些函数的实现本身就依赖于内核。

根据 ISO C 标准 [1]，freestanding environment（独立环境）是指不假定存在操作系统的运行环境。符合该环境要求的程序仅能使用 C 标准头文件的有限子集，即 float.h、iso646.h、limits.h、stdarg.h、stdbool.h、stddef.h 和 stdint.h。理想情况下，操作系统内核及其扩展应当符合独立环境标准。此外，由于通用操作系统被设计为可在多种硬件平台上运行，嵌入式语言解释器不应存在平台相关问题，例如 endianness（字节序）。

由于 portability（可移植性）是 Lua 的主要设计目标之一，其核心部分几乎完全符合独立环境标准，仅依赖少量额外的标准头文件。然而，这些依赖在源代码中易于追踪，因为它们被限制在单一的配置头文件（configuration header file）中。浮点相关代码同样被限制在该头文件中。进一步地，所有与操作系统相关的代码均被置于 Lua 核心之外的外部库中。例如，Lua 核心并不链接 C 内存分配函数；它通过调用在创建新解释器状态（interpreter state）时作为参数传入的函数来分配内存。Lua 使用纯 ISO C [1] 编写，不存在硬件平台依赖。

操作系统内核的另一项约束是体积。操作系统内核从系统启动到关闭一直驻留在内存中，因此内核大小是一个重要问题。操作系统内核的 binary image（二进制映像）通常小于 5 MB（常见的 Linux 发行版内核约为 3 MB）。与内核本身一样，嵌入式解释器也必须具有较小的体积并保持合理的效率。

与其他脚本语言相比，Lua 的 footprint（内存占用）非常小。Lua 5.1.4 独立解释器连同所有 Lua 标准库在 Ubuntu 10.10 上仅占 258 KB；而其他语言如 Python 和 Perl 则占用数兆字节^1，与完整操作系统内核处于同一数量级。

> ^1 在 Ubuntu 10.10 上，Python 2.6.5 占用 2.21 MB，Perl 5.10.1 占用 1.17 MB。

最后，由于操作系统内核拥有对全部硬件的无限制访问权限，必须对内核扩展施加特殊约束，以防止对系统资源的破坏或未授权访问。嵌入式语言必须提供某种手段来隔离扩展代码，并限制其对内核环境的访问。

Lua 提供了编程层面的支持以对脚本实施访问限制。与大多数脚本语言一样，Lua 具备 automatic memory management（自动内存管理）机制，可防止脚本通过指针直接操作内存。Lua 还支持 multiple state instances（多状态实例）；其 C 代码中完全不含全局变量。这一实现提供了完全的 state isolation（状态隔离），进而为将扩展彼此隔离以及与内核本身隔离提供了手段。由于可以为不同状态提供不同的库集合，因此能够创建具有不同 privilege levels（特权级别）的独立 protection domains（保护域）。

---

## 4. 基于 Lua 的内核脚本化环境

我们为内核脚本化设计的编程与执行环境由四个基本组件构成：正确嵌入内核的 Lua 解释器，用于执行 Lua 脚本；programming interface（编程接口），供内核开发者使其子系统支持脚本化；user interface（用户接口），用于向内核嵌入的 Lua 解释器加载和运行脚本；以及 Lua bindings（Lua 绑定），用于在内核与用户自定义脚本之间共享函数和数据结构。图 3 概述了该环境的架构与运行方式。

![[Pasted image 20260429202724.png]]

**图 3.** 基于 Lua 的内核脚本化架构与运行方式

### 4.1 运行概述

让我们以第 2.1 节中讨论的 CPUfreq 子系统的脚本化扩展为例。假设用户希望激活图 1 所示的 CPU 频率控制器。为此，她需要将该脚本加载到内核嵌入的 Lua 解释器中。为加载脚本，她使用用户接口（UI）提供的命令行工具，该工具允许她与 Lua 解释器进行动态交互。

脚本加载到内核后，CPUfreq 子系统通过其 embedding binding（嵌入绑定）周期性地调用函数 throttle，该函数实现了用户自定义的策略。除了允许内核子系统调用 Lua 脚本外，嵌入绑定还负责处理脚本执行过程中的错误。例如，如果函数 throttle 的执行失败，CPUfreq 的嵌入绑定可以调用默认例程来处理频率调节。

为获取当前 CPU 温度，函数 throttle 使用一个 extension binding（扩展绑定），该绑定通过函数 `acpi.get_temp` 提供温度信息。扩展绑定还允许脚本通过函数 `cpufreq.target` 设置 CPU 频率。函数 `get_load` 是一个用户自定义函数，同样通过扩展绑定获取信息以计算当前 CPU 负载。

内核扩展绑定通常以 LKM（Loadable Kernel Module，可加载内核模块）的形式开发，并作为常规 Lua 扩展库（以 C 语言实现）提供给脚本使用。它们可以通过内核子系统（经由内核编程接口）、用户接口（通过命令）或脚本自身（通过 Lua 标准库提供的 `require` 函数）加载到内核 Lua 状态中。

### 4.2 内核嵌入的 Lua

我们内核脚本化环境的核心组件是正确嵌入操作系统内核的 Lua 解释器。尽管将 Lua 嵌入 Linux 和 NetBSD 内核需要进行一些修改，但所有修改均为非侵入式的（non-intrusive），仅涉及修改 Lua 配置头文件中的若干宏，以及替换内核环境中不可用的 C 标准库中的部分功能设施。

我们所需进行的最重要的修改涉及浮点类型的使用。如前所述，操作系统内核不提供浮点类型支持。我们将 Lua 标准的 number 类型（定义为 double）替换为整数类型 intmax_t；这一修改仅需重新定义 luaconf.h 文件中的九个宏。我们选择 intmax_t 整数类型，是因为它能够方便地提供底层平台所支持的最大整数类型。

Lua 解释器在内存分配方面并不依赖 C 标准库，而是允许宿主程序提供自行实现的 memory allocator（内存分配器）。我们分别利用 Linux 和 NetBSD 内核中可用的内存分配原语（memory allocation primitive），实现了相应的分配器函数。两个内存分配函数的代码均不超过十八行。

我们还需进行的另一项修改是替换 Lua 用于 exception handling（异常处理）的 setjmp/longjmp 函数对。我们将这些函数替换为 Linux 和 NetBSD 内核中提供的等效函数。这一修改仅需重新定义 luaconf.h 文件中的三个宏。

除 Lua 解释器外，我们还嵌入了 Lua 基础库（basic library）以及若干不完全依赖操作系统资源或浮点类型的 Lua 标准库（debug 库、coroutine 库、table 库和 string 库）。我们所需做的唯一修改是：从基础库和 debug 库中移除部分依赖操作系统的功能，以及从 string 库中移除浮点格式。

### 4.3 用户接口

用户接口（User Interface, UI）由两部分组成：一部分运行在用户空间，另一部分运行在内核内部。用户层组件包括一个命令行工具和一个 pseudo-device descriptor file（伪设备描述符文件）。内核组件则是对应的 pseudo-device driver（伪设备驱动程序）。用户层工具类似于 Lua 的独立解释器（stand-alone interpreter），但它并非在用户空间中执行 Lua 脚本，而是在内核嵌入的 Lua 解释器中执行。

用户层命令接口实质上是伪设备驱动程序的前端。当用户发出命令时，UI 用户层组件通过调用 ioctl 系统调用（system call），将命令转发至伪设备驱动程序注册的 handler function（处理函数）。该处理函数在内核内部运行，提供用于管理内核 Lua 状态以及在这些状态中加载和运行脚本的实际命令。

伪设备驱动程序仅允许 privileged access（特权访问）；即它只处理由特权用户提交的请求。在处理任何从用户空间提交的命令之前，处理函数会检查用户凭证。若用户具有管理员权限，则命令被处理；否则，返回访问错误。

### 4.4 内核编程接口

为使某一内核子系统具备可脚本化能力，开发者需要实现适当的绑定，以在内核内部完成 Lua 的扩展和嵌入。在这两种情况下，开发者均需使用 Lua C API 和 KPI（Kernel Programming Interface，内核编程接口）。KPI 由上一节介绍的伪设备驱动程序实现。KPI 提供的功能包括：创建和销毁 Lua 状态、同步对这些状态的访问，以及向其中加载扩展库等。

在我们的脚本化环境中，Lua 脚本被加载到通过 KPI 创建的内核 Lua 状态中。这些状态与使用常规 Lua C API 创建的 Lua 状态类似，但它们针对内核环境进行了适配，需要使用 synchronization primitives（同步原语）和特殊的内存分配器。

新创建的内核 Lua 状态会在伪设备驱动程序中注册，以便可以通过 UI 命令从用户空间对其进行访问。由于系统调用处理函数与内核子系统在并发流中执行，它们对内核 Lua 状态的访问必须通过同步机制加以协调。如果某一 Lua 状态可被子系统内部的并发控制流访问，或在不同子系统间共享，同步同样是必要的。

内核 Lua 状态的同步基于底层操作系统内核提供的 mutual exclusion（互斥）机制。除存储常规的 Lua 执行状态外，内核 Lua 状态还包含一个互斥句柄。对状态的同步访问通过编程规约（programming discipline）来保证：在使用状态之前和之后，分别对其加锁和解锁。KPI 为这两种操作提供了同步原语。

扩展绑定必须在伪设备驱动程序中注册，以允许 Lua 脚本通过 `require` 函数动态加载这些绑定，其方式与加载常规 Lua 库相同。KPI 提供了该注册操作。

---

## 5. 评估

脚本语言在多个领域获得广泛采用的主要原因之一，在于系统性能与程序员生产力之间的权衡。由于这种权衡关系的存在，很难对使用脚本语言所带来的生产力提升进行客观评估 [28]。

对我们系统的客观评估同样面临这一困难。因此，在本节中我们并不试图论证内核脚本化优于（或劣于）其他操作系统扩展形式，而是论证内核脚本化是一种可行的扩展形式，可与其替代方案相媲美。

为评估我们的内核脚本化环境，我们为两个内核子系统实现了扩展：Linux 的 CPUfreq 子系统和 NetBSD 的 NPF（数据包过滤器）子系统。我们选择这两个子系统，是因为它们具有一些有利于脚本化扩展的特性。首先，它们已经提供了对扩展模块的支持，而这些扩展模块可以显著受益于在内核内部运行。其次，它们不提供操作系统核心服务，例如进程调度。因此，我们能够以非侵入式的方式开发扩展，而无需修改操作系统内核的核心组件。

我们的评估基于以下问题，这些问题源自第 2.2 节中讨论的需求：

- **实用性（Usefulness）：** 脚本是否实现了有用的扩展？
- **简洁性（Simplicity）：** 脚本是否易于开发？
- **性能（Performance）：** 脚本的执行是否足够高效？是否影响了系统整体性能？
- **可靠性（Reliability）：** 脚本是否影响了系统的可靠性？

我们的第一个实验是对 Linux CPUfreq 子系统的扩展。CPUfreq 支持以可加载内核模块的形式动态添加新的 CPU 频率控制器。我们利用此特性开发了一个 CPUfreq 模块，该模块支持以内核可加载用户脚本的形式动态添加新的频率控制器。我们的 CPUfreq 模块同时实现了嵌入绑定和扩展绑定：它周期性地调用一个 Lua 函数来调整 CPU 频率，同时向 Lua 代码暴露必要的内核资源。

在内核空间中执行 CPUfreq 控制器的主要原因是能够对 CPU 频率变更需求做出迅速响应。如果控制器在用户空间中执行，则难以满足此需求。例如，CPUfreq 内置控制器 Ondemand 根据 CPU 负载调整 CPU 频率。为了在需要更多处理能力时快速响应，Ondemand 控制器将其 rescheduling interval（重新调度间隔）设置为 CPU 频率切换延迟的二百倍。在我们的实验中，我们使用了一颗 Intel Pentium M CPU 1.6 GHz，其调度间隔约为 20 毫秒。

我们实现了 Ondemand 的 Lua 脚本版本 Ondemand.lua，使用了与原始控制器相同的重新调度间隔。Lua 版本的平均执行时间为 8 微秒，仅占其重新调度间隔的 0.04%。因此，使用 Lua 实现的 CPU 频率控制器并未引入可测量的开销。

我们的 Ondemand Lua 脚本版本共有 47 行 Lua 代码（包括使用扩展绑定访问 CPU 负载信息的 Lua 函数 `get_load`）。虽然它是原始控制器的简化版本，但其代码行数不到原始控制器（Linux 内核 2.6.25 上的 618 行 C 代码^2）的 8%。CPUfreq 的 Lua 绑定包含 138 行 C 代码，这使得我们的整体实现规模不到原始控制器的 30%。

> ^2 基于 Linux 内核 2.6.25。

Ondemand 的 Lua 版本还可以在运行时进行配置。例如，用户可以通过 UI 界面加载一个脚本来修改全局变量 `up` 的值，从而改变触发 CPU 频率提升的 CPU 负载阈值。用户还可以重新定义函数 `get_load`，使其仅考虑用户进程所消耗的 CPU 时间。在我们的实验中，我们使用 Ondemand 策略的 Lua 版本以便与 CPUfreq 原始内置控制器进行比较。然而，也可以开发其他有用的扩展，例如第 2.1 节中讨论的过热预防脚本。

与 CPUfreq 类似，NetBSD 的 NPF 子系统支持以可加载内核模块的形式动态添加新的数据包过滤规则程序。在我们的第二个实验中，我们开发了一个 NPF 模块，该模块支持以内核可加载用户脚本的形式添加新的数据包过滤规则。我们的 NPF 模块实现了一个嵌入绑定，允许该模块调用 Lua 函数来过滤网络流量；其扩展绑定向 Lua 代码暴露必要的内核资源。

应用于第二层到第四层的数据包过滤器通常在内核空间中实现，以避免对网络应用施加不当的延迟。通过内核可加载脚本实现过滤功能，我们也能够满足更高层检测的低延迟需求。我们的 NPF 扩展脚本用于过滤 SSH 流量以阻止对存在漏洞的服务器的访问；该脚本已在第 2.1 节的图 2 中展示。我们在一台运行于 Intel Core i3 CPU 3.10 GHz 上的虚拟机中执行该数据包过滤脚本。一条 100 Mbps 的虚拟网络将该虚拟机与其宿主机相连。使用 TCP 带宽测量工具 Iperf [10]，我们在两种场景下测量了网络带宽：启用脚本和未启用脚本。两种情况下的平均带宽均约为 96 Mbps。因此，使用 Lua 实现的数据包过滤规则并未引入可测量的开销。

SSH 协议过滤脚本共有 22 行 Lua 代码。仅使用 NPF 规则无法实现等效的过滤功能。嵌入绑定约有 200 行 C 代码。我们的过滤脚本使用了 Luadata，这是一个专门为我们的内核脚本化环境开发的 Lua 扩展库。Luadata 以安全的方式向 Lua 代码暴露内核内存，并允许 Lua 脚本将 data layouts（数据布局）应用于内存块，从而能够访问这些内存块中的划定字段。使用一种功能完备的语言，配合适当的库（如 Lua 字符串库和 Luadata），使我们能够轻松实现 SSH 过滤功能。

我们的内核脚本化环境仅允许特权用户在内核中加载和运行扩展脚本。因此，内核脚本对系统完整性的威胁不会超过可加载内核模块。我们的环境还限制了 Lua 解释器执行的指令数量，从而防止扩展脚本独占内核执行时间。嵌入绑定为每个扩展创建独立的 Lua 执行状态，提供扩展之间的隔离。扩展脚本还被置于沙箱中运行，即它们只能使用一组受限的扩展绑定。

独立的研究团队已使用 Lunatik——我们基于 Lua 的内核脚本化环境的首个实现——来进行操作系统内核扩展的实验。巴塞尔大学的计算机网络研究组开发了针对 Linux Netfilter 子系统的绑定，允许用户使用 Lua 脚本实现网络数据包处理功能 [11]。在他们的实验中，他们使用 Lua 实现了 NAT（Network Address Translation，网络地址转换），使用一台运行于 Intel Celeron CPU 2.4 GHz 上的路由器和一个 100 Mbps 的局域网，并测量了其 Lua NAT 实现与 Linux 内置 NAT 实现的最大吞吐量。两种情况下，测量的吞吐量均约为 90 Mbps。他们的分析还表明，Lua 脚本与内置实现具有近似相同的延迟和饱和区间。他们的 NAT 实现仅有 19 行 Lua 代码。

帕德博恩大学和美因茨大学的研究团队使用 Lunatik 为 pNFS 文件系统引入了脚本化功能，允许 pNFS 客户端使用 Lua 实现文件布局以配置存储策略 [12]。他们使用 Intel Xeon CPU 3.30 GHz 测量了多个文件布局脚本的执行时间。最简单脚本的平均执行时间为 1 微秒，功能完备的脚本为 8 微秒。在他们的实验中，一个简单的文件布局脚本仅有 8 行 Lua 代码。

上述两个研究团队均通过沙箱机制来保证系统可靠性，即通过限制向内核脚本暴露的扩展绑定来实现。

---

## 6. 相关工作

许多可扩展操作系统利用编程语言资源来提供可扩展性。其中一些系统，如 SPIN [6] 和 Singularity [13]，使用系统语言；SPIN 使用 Modula-3 的一个子集，Singularity 使用 C# 的一个扩展版本。大多数传统操作系统也属于此类情况，它们通常使用 C 语言来编写可加载内核模块。另一些系统则使用一组领域特定语言；Exokernel [8] 就是这样一个系统，它使用适用于特定任务（如创建硬盘和网络抽象）的受限语言。某些传统操作系统也属于此类情况，它们使用 DSL 来提供数据包过滤功能。

我们的方法与上述两类可扩展系统的区别在于所提供的可扩展性层次。通过脚本语言扩展操作系统内核，处于提供有限的领域特定语言与提供功能完备的系统编程语言之间的中间地带。由于可扩展性的层次与我们在第 2.2 节中讨论的问题密切相关，选择何种语言来扩展操作系统内核是这些因素之间的一种权衡。

与系统语言相比，脚本语言通常更易于开发扩展和实施保护。然而，它们通常在实用性和效率方面较为逊色。这些不足可以分别通过提供适当的绑定和应用优化技术来加以缓解。

与某些 DSL 相比，脚本语言通常更加实用和高效。另一方面，DSL 在开发扩展和实施保护方面可能更为容易。这些不足可以通过使用适当的绑定和应用沙箱技术来加以缓解。

脚本语言还具有提供通用语言核心（common language core）的优势，该核心可用于许多不同的领域。一旦为支持某个扩展（例如数据包过滤设施）而提供了内核脚本环境，它就可以在多种其他场景中被复用，既可用于编写其他扩展——设备驱动程序、网络协议、磁盘抽象——也可用于实验。此外，只拥有一个扩展语言引擎有助于简化保障系统完整性的任务。

除了使用系统语言和使用领域特定语言的可扩展系统之外，还有一个实际使用脚本语言的可扩展系统。µChoices 操作系统 [7] 提供了一种类似于 Tcl 的脚本语言来编写内核扩展，允许用户在其内核中加载和运行脚本。通过脚本，用户可以将系统调用聚合为批处理，以避免上下文切换，从而提高系统性能。扩展脚本被加载到内核嵌入式 Tcl 解释器的实例中，这些实例在独立的系统进程中执行 [22]。一组扩展绑定向脚本暴露扩展内核所需的资源。我们的方法与 µChoices 的区别在于，我们除了支持通过扩展语言解释器来实现脚本化之外，还支持通过嵌入语言解释器来实现脚本化。

最后，我们的方法与大多数以往可扩展操作系统的另一个重要区别在于，我们一直专注于通过内核脚本化来扩展现有的通用操作系统，而非从头开始实现一个完整的可脚本化操作系统。

---

## 7. 结论

在本文中，我们提出了 scriptable operating system（可脚本化操作系统）的概念，该概念源于将脚本化扩展的思想应用于可扩展操作系统的概念。基于这一概念，我们开发了一个脚本环境，允许通过用户脚本扩展内核子系统，这些脚本可以动态加载并在内核内部执行。

我们的内核脚本环境使用 Lua——一种流行的脚本语言——并仅进行了最少的修改。将 Lua 嵌入 Linux 和 NetBSD 内核的便捷性证明了其卓越的可移植性，表明它甚至可以在操作系统内核这样的苛刻环境中使用。

先前的工作已经探索了通过用户脚本扩展操作系统的思想；然而，其中大多数提供的是受限的领域特定语言，仅适用于特定任务。我们的环境则提供了一种通用的、功能完备的编程语言，这不仅允许开发更为复杂的扩展，还为内核开发者自身提供了一个有趣的编程环境。据我们所知，它也是第一个同时通过嵌入和扩展脚本语言来提供内核可扩展性的环境。此外，我们在现有的通用操作系统之上进行工作，而非在最初就为脚本化而设计的系统上。

我们已经在两个通用操作系统上实现了我们的脚本环境：Linux 和 NetBSD。我们的 NetBSD 实现现已成为 NetBSD 官方发行版的一部分。我们的第一个 Linux 实现 Lunatik 已被不同的研究小组用于支持内核子系统的扩展 [11, 12]。

对我们内核脚本实验的积极评估表明，我们提出的基于 Lua 提供操作系统可扩展性的方案是一种可行且有趣的操作系统内核扩展方法。我们相信这种方法可以促进操作系统开发的创新。首先，它为内核提供了一个更高层次的编程环境，支持便捷和敏捷的原型开发与实验。其次，它允许通常不具备内核编程技能的应用开发者根据自身需求对操作系统内核进行实验。

---

## 致谢

NetBSD 内核中的 Lua 部分是作为 Google Summer of Code 项目开发的，由 The NetBSD Foundation 赞助。我们也感谢 NetBSD 开发者给予的所有支持。在 Lunatik 的开发过程中，Lourival Vieira Neto 获得了 CAPES（巴西高等教育改进机构）的资助，Roberto Ierusalimschy 获得了 CNPq（巴西研究委员会）的资助。

---

## 参考文献

[1] ISO C standard 1999. Technical report, 1999. URL http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1124.pdf. ISO/IEC 9899:1999 draft.

[2] IEEE Standard for Local and Metropolitan Area Networks—Station and Media Access Control Connectivity Discovery. IEEE Standard 802.1AB, 2009.

[3] G. Banga, P. Druschel, and J. C. Mogul. Resource containers: A new facility for resource management in server systems. In OSDI, volume 99, pages 45–58, 1999.

[4] L. P. Barreto, R. Douence, G. Muller, and M. Südholt. Programming os schedulers with domain-specific languages and aspects: New approaches for os kernel engineering. In Proceedings of the 1st AOSD Workshop on Aspects, Components, and Patterns for Infrastructure Software, pages 1–6. Citeseer, 2002.

[5] M. Belshe and R. Peon. SPDY protocol—Draft 3.1. URL http://www.chromium.org/spdy/spdy-protocol/spdy-protocol-draft3-1.

[6] B. Bershad, C. Chambers, S. Eggers, C. Maeda, D. McNamee, P. Pardyak, S. Savage, and E. Sirer. SPIN—an extensible microkernel for application-specific operating system services. ACM SIGOPS Operating Systems Review, 29(1):74–77, 1995. ISSN 0163-5980.

[7] R. H. Campbell and S.-M. Tan. µChoices: an object-oriented multimedia operating system. In Proceedings of the Fifth Workshop on Hot Topics in Operating Systems, 1995 (HotOS-V), pages 90–94. IEEE, 1995.

[8] D. Engler. The Exokernel operating system architecture. PhD thesis, MIT, 1998.

[9] R. Fielding, J. Gettys, J. Mogul, H. Frystyk, L. Masinter, P. Leach, and T. Berners-Lee. Hypertext Transfer Protocol—HTTP/1.1, 1999.

[10] M. Gates, A. Warshavsky, A. Tirumala, J. Ferguson, Jim Dugan, F. Qin, K. Gibbs, J. Estabrook, A. Gallatin, S. Hemminger, J. Nathan, and G. Renker. Iperf, 2008. URL http://iperf.sourceforge.net.

[11] A. Graf. PacketScript—a Lua Scripting Engine for in-Kernel Packet Processing. Master's thesis, Computer Science Department, University of Basel, July 2010.

[12] M. Grawinkel, T. Suss, G. Best, I. Popov, and A. Brinkmann. Towards Dynamic Scripted pNFS Layouts. In High Performance Computing, Networking, Storage and Analysis (SCC), 2012 SC Companion:, pages 13–17. IEEE, 2012.

[13] G. Hunt and J. Larus. Singularity: rethinking the software stack. ACM SIGOPS Operating Systems Review, 41(2):37–49, 2007. ISSN 0163-5980.

[14] R. Ierusalimschy. Programming in Lua. Lua.org, third edition, 2013.

[15] R. Ierusalimschy, L. de Figueiredo, and W. Celes Filho. Lua—an extensible extension language. Software: Practice and Experience, 26(6):635–652, 1996.

[16] R. Ierusalimschy, L. de Figueiredo, and W. Celes. Lua 5.1 Reference Manual. Lua.org, first edition, 2006.

[17] R. Ierusalimschy, L. de Figueiredo, and W. Celes. The evolution of Lua. In Proceedings of the third ACM SIGPLAN Conference on History of Programming Languages, pages 2–26. ACM, 2007.

[18] R. Ierusalimschy, L. H. de Figueiredo, and W. Celes. Passing a language through the eye of a needle. Communications of the ACM, 54(7):38–43, 2011.

[19] M. F. Kaashoek, D. R. Engler, G. R. Ganger, and D. A. Wallach. Server operating systems. In Proceedings of the 7th Workshop on Systems Support for Worldwide Applications (ACM SIGOPS European workshop), pages 141–148. ACM, 1996.

[20] B. Lampson. On reliable and extendable operating systems. In Proceedings of the Second NATO Conference on Techniques in Software Engineering, 1969.

[21] G. Lefkowitz. Extending vs. embedding—there is only one correct decision, 2003. URL http://twistedmatrix.com/users/glyph/rant/extendit.html.

[22] Y. Li, S.-m. Tan, M. L. Sefika, R. H. Campbell, and W. S. Liao. Dynamic Customization in the µChoices Operating System. In Proceedings of Reflection'96, 1996.

[23] M. K. McKusick, K. Bostic, M. J. Karels, and J. S. Quarterman. The design and implementation of the 4.4 BSD operating system. Pearson Education, 1996.

[24] P. Mochel and M. Murphy. sysfs—The filesystem for exporting kernel objects, 2009. URL http://kernel.org/doc/Documentation/filesystems/sysfs.txt.

[25] H. Muhammad and R. Ierusalimschy. C APIs in extension and extensible languages. Journal of Universal Computer Science, 13(6):839–853, 2007.

[26] J. Ousterhout. Scripting: Higher-level programming for the 21st century. IEEE Computer, 31(3):23–30, 1998.

[27] V. Pallipadi and A. Starikovskiy. The ondemand governor. In Proceedings of the Linux Symposium, volume 2, pages 215–230, 2006.

[28] L. Prechelt. An empirical comparison of seven programming languages. Computer, 33(10):23–29, 2000.

[29] M. Rasiukevicius. NPF—Progress and Perspective. AsiaBSDCon 2014, page 21, 2014.

[30] S. Savage and B. Bershad. Issues in the design of an extensible operating system. In Proceedings of the First USENIX Symposium on Operating Systems Design and Implementation (OSDI), 1994.

[31] H. Schulzrinne, S. Casner, R. Frederick, and V. Jacobson. RTP: A Transport Protocol for Real-Time Applications, 2003.

[32] M. I. Seltzer, Y. Endo, C. Small, and K. A. Smith. Dealing with disaster: Surviving misbehaved kernel extensions. In OSDI, volume 96, pages 213–227, 1996.

[33] M. I. Seltzer, Y. Endo, C. Small, and K. A. Smith. Issues in extensible operating systems. Computer Science Technical Report TR-18-97, Harvard University, 2(2.2):1, 1997.

[34] C. Small and M. Seltzer. VINO: An Integrated Platform for Operating System and Database Research. Technical report, 1994.

[35] T. Voigt, R. Tewari, D. Freimuth, and A. Mehra. Kernel mechanisms for service differentiation in overloaded web servers. In USENIX Annual Technical Conference, General Track, pages 189–202, 2001.

[36] T. Ylonen and C. Lonvick. The Secure Shell (SSH) Transport Layer Protocol, 2006. URL http://www.ietf.org/rfc/rfc4253.txt.

<!-- obsidian-publish-source: Operating System 操作系统/Pappers 论文/Others/基于 Lua 的可脚本化操作系统.md -->