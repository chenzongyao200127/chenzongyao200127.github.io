**Andrew Ferraiuolo**（康奈尔大学）
**Andrew Baumann**（微软研究院）
**Chris Hawblitzel**（微软研究院）
**Bryan Parno**（卡内基梅隆大学）

---

## 摘要

Intel SGX 承诺提供强大的安全性：任意数量的用户态飞地（enclave）可以抵御物理攻击和特权软件对手。然而，为了实现这一点，Intel 将 x86 架构扩展了一种复杂度接近操作系统微内核的隔离机制，由不透明的硅片与微码混合实现。虽然基于硬件的安全性能够提供纯软件难以或无法实现的性能和功能，但纯硬件解决方案难以更新——无论是修补安全缺陷还是引入新功能。

Komodo 展示了一种实现经认证的、按需的、用户态的、并发隔离执行的替代方案。我们将核心硬件机制（如内存加密、地址空间隔离和远程证明）与其管理逻辑解耦，Komodo 将管理工作委托给一个特权软件监视器（monitor），由该监视器实现飞地。监视器的正确性通过机器可检查的证明来保证，证明内容涵盖功能正确性以及飞地完整性和机密性的高层安全属性。我们通过在 ARM TrustZone 上用经过验证的汇编代码实现的具体原型，证明了该方案的实用性和高性能。我们的最终目标是实现等同于或优于 SGX 的安全性，同时使新的飞地特性能够独立于 CPU 升级进行部署。

Komodo 的规约、原型实现和证明可在 https://github.com/Microsoft/Komodo 获取。

---

## 1. 引言

软件保护扩展（SGX）[43] 是近年来 Intel CPU 中用于软件飞地强隔离的一组新指令。与之前主流的可信计算硬件 [2, 83] 相比，SGX 包含强物理安全性（内存加密），并支持多个飞地：任意数量的飞地可以并发运行，而无需信任内核或虚拟机管理程序。尽管如此，SGX 仍为飞地代码支持熟悉的用户态执行环境，并保持与现有操作系统和虚拟机管理程序的兼容性。自 SGX 规范发布以来的短短时间内，已经涌现出大量应用 [例如 5, 8, 9, 14, 21, 42, 65, 68, 75]，竞争厂商也在开发类似功能 [48]。

然而，与大多数新硬件一样，SGX 的部署和演进一直很慢。SGX 版本 2 引入了对许多飞地应用至关重要的动态内存管理功能 [5, 8, 42]，于 2014 年 10 月发布规范 [43]，但三年后仍没有将要实现它的 CPU 的公告。硬件开发的缓慢步伐并非新鲜事，但 SGX 在 CPU 特性中几乎是独一无二的，因为它没有替代方案——其他架构扩展提升性能，但通常可以使用传统指令实现等效功能。此外，正如我们在第 2 节中详述的，SGX 的安全性依赖于微码和硅片中不透明的实现 [18]，且已经存在已知缺陷，包括安全漏洞 [61] 和 " 受控信道 "（controlled-channel）攻击——后者利用操作系统诱发和观察飞地页错误的能力来泄露飞地数据 [78, 88]。

鉴于硅片缩放的放缓趋势 [22, 79]，将飞地等关键安全特性绑定到硬件实现是危险的。每一次增量变更——例如纠正受控信道等安全缺陷，甚至添加动态分配等功能——都必须等待新 CPU 的部署，因此需要很多年。软件本质上比硬件更具可塑性，两者之间有效的分工将允许独立于新硬件来开发新功能和修复缺陷。硬件厂商可以简化其 CPU 的复杂性 [7]，降低验证工作量和缺陷风险，并专注于提升内存加密等硬件功能的容量和性能。

在本文中，我们的目标是将飞地的管理与底层硬件机制（如保护、证明和内存加密）解耦。我们的核心观察是：SGX 的安全属性并不依赖于它完全作为 CPU 特性来实现。类似的隔离机制自第一代多用户系统以来就已存在；SGX 的与众不同之处在于内存加密、不依赖大型不可信操作系统，以及 " 硬件比软件更安全 " 的民间直觉。

Komodo 借鉴了 SGX 的思想，但用形式化验证取代了民间直觉。与 SGX 一样，它依赖硬件支持的内存加密和地址空间隔离。然而，Komodo 不使用飞地操控指令，而是以经过验证的汇编代码实现为软件引用监视器。实际上，Komodo 的设计反映了 SGX 内部在核心硬件与指令集之间的分离。由于监视器的唯一角色是保护飞地，它比完整的经过验证的内核 [35, 49, 89] 要简单得多（因而更容易演进）。它也可以很容易地更新。例如，在开发了模仿 SGXv1 的 Komodo 初始版本之后，我们添加了类似于 SGXv2 的动态内存管理。我们在大约 6 个人月内实现并验证了这次更新。

我们在第 4 节详细描述 Komodo。除了在第 5 节中形式化其规约外，我们在第 6 节中证明了它保护飞地的机密性和完整性免受其他飞地和不可信操作系统的侵害。据我们所知，没有其他安全飞地实现提供此类形式化保证。该证明适用于规约的任何正确实现，包括我们在第 7 节中描述并在第 8 节中评估的基于 ARM TrustZone 的原型。

Komodo 不支持多处理器执行——虽然操作系统可以在多个核心上运行，但监视器和飞地被限制在单个核心上。低级并发代码的验证仍然具有挑战性（尽管近期取得了进展 [35]），ARM 的弱内存一致性 [54] 增加了额外的复杂性。我们将此留作未来工作。

我们工作的贡献包括：

- 对硬件需求的识别（第 3 节）和在软件中实现飞地的设计（第 4 节）；
- ARMv7 实质性子集的形式化模型，包括用户模式和特权模式、TrustZone、页表和异常（第 5.1 节）；
- Komodo 的高层形式化功能规约（第 5.2 节），以及保证飞地程序机密性和完整性的证明，形式化为无干扰性（noninterference）（第 6 节）；
- 经过验证的原型（第 7 节）和评估（第 8 节），显示性能可与 SGX 竞争；
- 支持 " 经过验证的软件可以比硬件演进更快 " 这一假说的证据（第 7.3 节）。

---

## 2. 背景与动机

此前的研究工作专注于使用硬件机制保护关键软件，即使特权软件（如操作系统或虚拟机管理程序）是恶意的或被攻陷的 [15, 16, 19, 26, 50, 57, 67, 82]。SGX 是此类保护的首次商业尝试，其设计部分受到务实实现约束的驱动，例如与现有操作系统资源管理机制的兼容性，以及避免对处理器快速路径的修改 [18]。本节提供 SGX 设计的高层概述，以便为我们自身设计提供参考。

SGX 的实现由三个组件构成：(i) 由内存控制器中的加密引擎实现的对静态物理内存区域的加密和完整性保护；(ii) 一组允许创建、操控和执行飞地的指令；(iii) 对处理器 TLB 缺失和异常处理流程的修改，以对加密内存区域的访问施加飞地保护。

SGX 采取的基本方法是充当不可信操作系统和/或虚拟机管理程序（我们统称为 "OS"）所执行操作的引用监视器。尽管 OS 没有对加密页的直接访问权限，但它会将加密页分配并映射到飞地中；尽管它不能直接操控飞地的寄存器状态，但 OS 决定何时以及在哪些 CPU 上执行飞地线程。OS 通过 SGX 指令间接管理飞地页，这些指令操控飞地页缓存映射（EPCM）——一个维护在加密内存中且软件不可访问的数据结构。EPCM 存储每个加密页的元数据，包括其分配状态、类型、所属飞地、权限和虚拟地址。EPCM 实质上是加密页的反向映射，在 TLB 缺失时也会被查询以强制执行飞地的内存保护——每个页表映射必须与 EPCM 一致。

由于 SGX 指令更新的是一个复杂的数据结构，因此这些指令本身也很复杂。例如，除了基本的参数有效性检查外，EADD 指令必须检查要添加到飞地中的页是空闲的且飞地处于正确状态，然后更新新页的 EPCM 条目和飞地控制结构。在此过程中，它必须防范页的并发分配或飞地的并发修改。其他 SGX 指令具有更大的复杂性——例如在回收 EPC 页之前验证 TLB 刷新的过程涉及飞地控制结构中维护的一系列纪元计数器。这些指令通常也不是性能关键的；实际上，根据 Intel 的专利 [46, 60]，Costan 和 Devadas[18, §2.14] 声称它们完全由微码实现。

无论这些指令如何实现，SGX 的安全性取决于它们的正确性。Intel 已发表了使用 SMT 求解器对高层 SGX 模型进行形式化验证的细节 [31, 44]，并已验证了并发 SGX 操作（不同）模型的可线性化性 [52]。然而，这些模型与 SGX 实现之间似乎不存在形式化联系，而实现中至少有一个安全漏洞已被修补 [61]。可以预期会发现更多缺陷，正如在过去的 CPU 安全技术中发生的那样 [86, 87]。

即使假设实现正确，SGX 仍然容易受到多种攻击。经典的侧信道攻击利用共享硬件资源（如缓存、分支预测器和 TLB），且 SGX 未予应对 [13, 76]。飞地必须转而使用（通常代价高昂的）缓解措施，如避免依赖于秘密的内存访问。Schwarz 等人 [76] 观察到，现有的缓存分区硬件支持（Intel 缓存分配技术 [63]）如果在飞地进入时被激活，可以击败此类攻击，但这并非 SGX 目前提供的功能。我们将高层飞地实现从硬件中解耦的目标之一，就是允许此类修复独立于硬件或架构变更进行部署。

除了经典侧信道外，飞地还容易受到新型 " 受控信道 " 攻击，其中操作系统利用其诱发和观察飞地页错误的能力来推断秘密 [78, 88]。缓解措施存在，但至少需要重新编译飞地代码、阻止使用动态分页，且带来高昂的性能代价 [77, 78]。

总体而言，我们认为 SGX 的局限性是系统性的：由于其整个规范是 x86 架构的一部分，SGX 在添加功能或应对威胁方面简直太慢了，而且硬件实现的局限性也束缚了其功能 [7]。在下一节中，我们描述如何将支持飞地的硬件与软件解耦，使两者能够独立演进。

---

## 3. 威胁模型与硬件

### 3.1. 威胁模型

与 SGX 一样，我们寻求保护在飞地内执行的用户态代码的机密性和完整性，使其免受完全控制平台特权软件（操作系统和虚拟机管理程序）的攻击者的侵害。为了保持跨平台的通用性，我们考虑此威胁模型的两种变体，取决于对内存的物理攻击是否在范围之内。

我们假设一个控制特权软件的软件攻击者。我们还信任我们的验证工具（Dafny 和 Z3，后文第 5 节描述）、汇编器、链接器和引导加载程序。在硬件方面，我们假设 CPU 是正确的。攻击者可以注入外部中断，并试图干扰 I/O 设备。如果物理内存攻击在范围之内，攻击者可以访问 CPU 封装外部的任何 RAM。这包括总线嗅探和冷启动 [36] 攻击。

与 SGX 一样，通用侧信道攻击不在范围之内。硬件隔离技术如缓存分区 [19, 63] 是以实用方式击败通用代码中此类攻击所必需的，我们预期 Komodo 的未来版本可以启用它们。Komodo 对受控信道攻击 [88] 免疫；正如我们的机密性证明（第 6 节）所保证的，操作系统仅能获知所发生异常的类型，且无法诱发异常。

### 3.2. 硬件需求

我们在 Komodo 中的基本方法是用经过验证的汇编代码实现一个高特权程序；其角色类似于 SGX 中的飞地管理指令：通过充当飞地操控和执行的引用监视器，维护一个类 EPCM 的安全页 " 数据库 "。

为了在软件中实现飞地，我们依赖四个硬件原语：为监视器代码/数据和飞地页提供的隔离内存、为特权监视器和非特权飞地提供的受保护执行环境、用于证明的信任根，以及随机数源。

**隔离内存。** Komodo 需要一个物理内存区域，其机密性和完整性由硬件保护。我们的设计对确切的内存隔离机制是不可知的，我们预期它将在不同应用中根据硬件威胁边界而有所不同。

如果物理内存攻击在范围之内，硬件必须包含内存加密和/或片上 RAM。例如，SGX 对 RAM 执行加密和完整性保护。这提供了对物理攻击的强保护，代价是有限的大小和完整性保护的性能惩罚 [44]。不幸的是，Intel 的内存加密引擎仅可由 SGX 访问，因此 Komodo 无法利用它。IBM SecureBlue[11] 也包含内存加密硬件，但公开信息有限，难以确定 IBM 的设计是否适用。AMD 最近发表了一个可由特权软件配置的硬件内存加密提案 [48]。由于该提案缺乏完整性保护，它可以扩展到大内存，但安全性较弱。

作为加密的替代方案，某些 " 片上系统 "（SoC）包含暂存 RAM（scratchpad RAM），它凭借其片上位置可以抵御大多数物理攻击 [41, 64]。虽然大小有限，但对于安全嵌入式应用可能很有效，因为它避免了加密的复杂性和能耗/性能开销 [17]。

最后，如果物理内存攻击不在范围之内（这在许多 SGX 之前的应用中很常见），硬件所需的只是一个类 IOMMU 的过滤器来对 RAM 进行分区并防止非特权软件或设备的访问。

**监视器的特权环境。** Komodo 的监视器必须受到保护，免受平台上恶意特权代码（包括操作系统和虚拟机管理程序）的侵害。此需求包括监视器代码与正常执行之间的安全控制转移机制，以及防止监视器代码内的非预期控制转移或对其中间状态（例如寄存器）的访问。注意，我们不假设另一层（代价高昂的）内存翻译——监视器需要访问隔离内存，但这可以采取直接物理映射或受限段的形式，否则不可访问。

多种架构已经包含这样的环境。在 SGX 中，它实际上由微码引擎提供：处理器保证微码序列的执行是不可中断的且受保护不受干扰。超级特权受限环境的其他例子包括 DEC Alpha PALcode[24] 和 RISC-V 机器模式 [85]。我们的原型利用了 ARM TrustZone 的安全监视器模式，下文将予描述。

**飞地执行环境。** Komodo 必须能够控制飞地的执行，保护自身和平台上的其他代码免受恶意飞地的侵害。典型的用户模式就足够了，如果它也能受到操作系统的保护的话。SGX 飞地的一个独特功能是我们不尝试模拟的虚拟内存布局。SGX 飞地在不可信用户进程的上下文中执行——飞地区域外的内存透明地反映其父进程的地址空间。相反，Komodo 在其自己的虚拟地址空间中执行每个飞地，与不可信进程共享的内存必须通过显式映射建立。在我们的经验中，很少有应用需要从飞地到不可信父进程的不受控制的临时访问。这样的功能不仅需要额外的硬件支持，还可以说降低了整体飞地安全性——因为飞地的大部分虚拟内存空间是不可信的，而在我们的模型中整个虚拟空间都是可信的（明确定义的共享映射除外）。

**远程证明。** Komodo 需要一个硬件支持的远程证明信任根。与基于 TPM 的信任链类似，我们的预期是硬件或早期引导加载程序将证明监视器的安全哈希（监视器继而实现飞地证明）。此类证明机制有广泛支持 [70, 83]。

**随机数源。** 最后，我们需要一个硬件支持的密码学安全随机数源。这可以是一条指令（如 x86 的 RDRAND/RDSEED[45]）或监视器可访问的硬件设备。

我们注意到 Sanctum[19]（一个修改后的 RISC-V CPU）满足上述所有需求，但需注意物理攻击不在其范围之内。事实上，Sanctum 与 Komodo 有很多共同点，但有几个关键差异：它依赖于重大的硬件修改（每飞地页表）；它要求比 Komodo 所采用的远为复杂的证明机制，且其安全保证依赖于用 5000 行 C++ 实现的未经验证的监视器的正确性。在本文中，我们对在 ARM TrustZone 上实现的此类安全监视器进行形式化验证。我们在第 10 节进一步讨论 Sanctum。

### 3.3. ARM TrustZone

我们在 TrustZone[2] 上实现 Komodo 原型，这是许多 ARM CPU 中的一种安全技术。TrustZone 扩展了核心架构和外设（如内存控制器）。

如图 1 所示，TrustZone 处理器运行在两个世界之一：运行常规操作系统和应用程序的正常世界（normal world），以及安全世界（secure world）。每个世界都包含用户模式和五种不同的特权模式；后者用于不同的异常（例如，页错误进入与系统调用不同的模式），但都具有同等特权。安全世界有第六种特权监视器模式，用于切换世界：正常世界中的安全监视器调用（SMC）指令可导致在监视器模式下发生异常。

某些系统控制寄存器是分组的（banked），每个世界各有一份。这包括 MMU 配置和页表基址寄存器，因此世界切换可能进入不同的地址空间。TLB 和缓存条目也按世界标记。I/O 设备也可以参与 TrustZone。页表条目中的安全位用于标记由安全世界代码发出的内存引用，且 TrustZone 感知的内存控制器或 IOMMU 等设备可以使用此标记防止正常世界访问安全世界的内存或设备 [1, 4]。世界之间内存的可访问性取决于特定的平台配置；例如，在某些 SoC 上，安全世界对隔离内存区域拥有独占访问权 [17, 70]。

我们选择在 TrustZone 上实现 Komodo 原型，因为其安全世界满足我们同时执行监视器和飞地代码的需求：飞地在安全世界用户模式下运行，使用由运行在安全特权模式（主要是监视器模式）下的 Komodo 建立的页表。此外，ARM 生态系统目前缺乏类飞地的功能；现有的 TrustZone 应用要么假设所有安全世界代码都是可信的 [6, 30, 47]，要么依赖基于语言的隔离来保护 "trustlet"[74]。

---

## 4. Komodo 设计与 API

Komodo 监视器基于上一节描述的硬件来实现飞地。与 SGX 一样，它管理一个隔离物理内存区域，使安全页可用于构建飞地，并在保护飞地内部状态的同时启用飞地执行。

表 1 中的 API 调用对应 SGX 操作，但它们不是不同的指令，而是作为监视器调用来调用的。

**表 1 — OS 和飞地调用 Komodo 监视器的 API**

**安全监视器调用（SMC，来自 OS）：**

| 调用 | 说明 |
|------|------|
| `GetPhysPages()→int npages` | 返回安全页的数量 |
| `InitAddrspace(PageNr asPg, PageNr l1ptPg)` | 给定两个空页，创建地址空间（飞地） |
| `InitThread(PageNr asPg, PageNr threadPg, void *entry)` | 创建线程 |
| `InitL2PTable(PageNr asPg, PageNr l2ptPg, int l1index)` | 分配二级页表 |
| `AllocSpare(PageNr asPg, PageNr sparePg)` | 为给定地址空间分配备用页 |
| `MapSecure(PageNr asPg, PageNr dataPg, Mapping va, InsecurePg content)` | 分配数据页，映射到 va 中的地址和权限 |
| `MapInsecure(PageNr asPg, Mapping va, InsecurePg target)` | 将非安全（共享）页映射到 va 中的地址和权限 |
| `Finalise(PageNr asPg)` | 标记飞地为最终态，计算度量值并允许执行 |
| `Enter(PageNr thread, int arg1, int arg2, int arg3)→int retval` | 在空闲线程上进入飞地，传递参数 |
| `Resume(PageNr thread)→int retval` | 恢复先前挂起的线程的执行 |
| `Stop(PageNr asPg)` | 标记飞地为已停止，允许释放 |
| `Remove(PageNr pg)` | 释放已停止飞地中的任何页或任何飞地中的备用页 |

**超级调用（SVC，来自飞地）：**

| 调用 | 说明 |
|------|------|
| `GetRandom()→u32 val` | 硬件安全随机数源 |
| `Attest(u32 data[8])→u32 mac[8]` | 构造飞地身份的证明 |
| `Verify(u32 data[8], u32 measure[8], u32 mac[8])→bool ok` | 检查证明的有效性 |
| `InitL2PTable(PageNr sparePg, int l1index)` | 从备用页创建二级页表 |
| `MapData(PageNr sparePg, Mapping vaddr)` | 将备用页映射为零填充数据页 |
| `UnmapData(PageNr dataPg, Mapping vaddr)` | 取消映射数据页，使其重新成为备用页 |
| `Exit(int retval)` | 将控制权返回给 OS |

**页类型与飞地构建。** 监视器必须确保安全页的一致使用，例如防止互不信任的飞地之间的双重映射。Komodo 使用一个我们称为 PageDB 的数据结构跟踪安全页的状态。这大致等同于 SGX 的 EPCM；对于每个安全页，它存储页的分配状态，以及（如果已分配的话）其类型和对所属飞地的引用。监视器不进行自身的分配——操作系统必须选择它已知为空闲的页，否则 API 调用将失败。

每个已分配的页属于六种类型之一：地址空间、线程、一级页表、二级页表、数据页和备用页。一个飞地由一个地址空间和至少一个线程组成。要开始构建飞地，操作系统调用 `InitAddrspace` 创建新的（空）地址空间。然而，在填充地址空间之前，操作系统必须使用 `InitL2PTable` 分配二级页表。Komodo 的 API 编码了两级层次页表，粒度选择反映了 ARM 的硬件页表格式。操作系统可以分配任意多个二级表，但要在给定虚拟地址处成功进行映射调用，相关的页表必须存在。

然后操作系统可以通过映射一个或多个安全和非安全数据页来填充地址空间。安全数据页位于隔离内存内，且对飞地是私有的。它们的初始内容、虚拟地址和页权限包含在下文描述的证明度量中。非安全页不受硬件隔离保护，因此不可信的操作系统可以访问。这些页可以映射到飞地中以促进与操作系统或飞地之间的不可信通信通道。

为使飞地可执行，操作系统还必须创建一个线程，指定其入口点地址。然后飞地被显式地最终化（finalise），在执行之前阻止进一步不受控的页/线程映射。

**飞地执行。** 属于已最终化飞地的新创建线程可通过调用 `Enter` 来执行，这会导致监视器切换到安全世界用户模式，并在线程的入口点地址处以给定参数开始执行。飞地线程随后执行直到发生异常：要么是中断，要么是飞地触发的异常（如页错误、未定义指令或系统调用）。在中断发生时，监视器在线程页中保存寄存器上下文，然后向操作系统报告中断。线程上下文被标记为已进入，以防止挂起的线程被重新进入。

飞地也可以通过超级调用（SVC）指令调用监视器。其中一个调用 `Exit` 用于显式地将结果传回操作系统。在这种情况下，飞地的寄存器不会被保存，允许其被重新进入。

如果飞地发生异常，线程仅以错误代码退出（不提供其他信息，以避免侧信道泄露）。与 SGX[88] 不同，操作系统不能诱发飞地页错误。因此我们的设计是安全的，也对不模拟非法指令或不处理页错误的简单飞地来说是充分的；我们预计在未来工作中添加飞地处理自身错误的机制。

**证明（Attestation）。** Komodo 采用了一种极简的证明设计，灵感来自先前关于本地证明的工作 [40, 58]。这一重要的设计选择使我们能够形式化验证证明机制，这对于更复杂的方案 [19] 将是具有挑战性的。

在飞地被构建的过程中，监视器构造页分配调用序列及其参数的哈希；具体来说：(i) 每个安全页的飞地虚拟地址、权限和初始内容；以及 (ii) 每个线程的入口点。与 SGX 一样，操作系统可以任意构建飞地，但飞地布局的任何变化都会反映在哈希中。当飞地被最终化时，此哈希成为飞地用于证明目的的不可变度量值。

与 SGX 一样，Komodo 将本地（同一机器）证明实现为监视器原语，并将远程证明推迟给一个可信飞地（我们尚未实现）。Komodo 的证明是使用在启动时从密码学安全随机数源生成的秘密密钥计算的消息认证码（MAC）。MAC 的计算覆盖 (i) 证明飞地的度量值，以及 (ii) 飞地提供的数据——后者可用于将公钥对绑定到飞地，从而引导与飞地外部代码的加密通信 [56]。监视器提供调用供飞地创建和验证证明。

**动态分配。** Komodo 包含对飞地内存动态管理的支持，可与 SGXv2[43] 相媲美。操作系统可以在任何时候使用 `AllocSpare` 监视器调用向飞地分配备用页。这些页不改变飞地的度量值，因为它们在飞地发出 `MapData` 或 `InitL2PTable` SVC 将其映射为数据页或页表之前不会变得可访问。飞地也可以取消映射数据页（将其变回备用页），操作系统可以回收备用页。因此，操作系统可以推断出备用页已被分配（因为尝试移除它们将会失败），但它无法判断飞地是否已将它们用作数据页或页表页。这与 SGXv2 形成对比，在 SGXv2 中操作系统仍然控制所有动态分配的类型、地址和权限。我们没有意识到针对此侧信道的攻击，但仍然看不出镜像它的理由。

**释放。** 在飞地的页被释放之前，操作系统必须调用 `Stop`。这会阻止进一步执行，并允许使用 `Remove` 释放安全页。地址空间进行引用计数，且必须最后被移除。

---

## 5. 规约

我们使用 Dafny[51]（一种通用验证语言）来规约和验证 Komodo。本节描述我们信任的 ARM 汇编语言 Dafny 规约（第 5.1 节）和 Komodo 整体正确性的规约（第 5.2 节）。我们通过证明若干高层引理来增强对 Komodo 规约的信心：规约维护页状态的一致性不变量（第 5.2 节描述），以及保证飞地机密性和完整性（第 6 节）。Dafny 在 Z3 SMT 求解器 [23] 的帮助下检查这些引理的有效性。

### 5.1. ARM 机器模型

我们的硬件规约用 Dafny 编写，涵盖 ARMv7 架构 [3] 的一个子集。我们将执行建模为一系列机器状态，其中状态包括机器的所有可见信息（如寄存器和内存）。我们的模型包括核心寄存器 R0-R12、栈指针（SP）、链接寄存器（LR）、当前和保存的程序状态寄存器（CPSR 和 SPSR）的部分、特权模式、控制流、中断和异常。我们建模了 25 条指令的语义，包括整数和位运算，以及对内存和控制寄存器的访问。

目前，我们的模型必须被完全信任，但这可以通过证明其与另一个 ARM 形式化模型 [29, 71, 72] 的正确性一致来避免。为帮助确保可信度，我们采用了 Ironclad[38] 所称的惯用规约方法论：我们仅规约 Komodo 实现所需的特性，且编写规约使得实现不能触发其他未规约的行为。例如，经过验证的实现不能执行未规约的指令。

为简化关于控制流的推理，我们不显式建模程序计数器（PC）寄存器。相反，我们的模型编码了有限形式的结构化控制流：if 语句、while 循环和子过程调用。这避免了关于 PC 更新和条件分支等控制流指令效果的验证负担。然而，我们确实建模了对 Komodo 正确性至关重要的两个控制转移的副作用：从特权代码到用户模式的分支（`MOVS PC, LR` 指令，它分支到链接寄存器并更新模式和标志），以及异常发生时切换回特权模式（保留异常前 PC 值在 LR 中）。因此 Komodo 规约可以使用其值隐式引用异常发生时的 PC。

32 位 ARM 架构包含我们也建模的寄存器分组（banking）特性：SP、LR 和 SPSR 寄存器根据当前模式分组——用户模式对 SP 的访问指向具体寄存器 SP_usr，而监视器模式代码访问 SP_mon 等。我们建模所有分组寄存器，但 FIQ 模式下独有的分组寄存器除外（不需要）。

**内存。** 在设计内存模型时，我们做了若干被证明对构建可扩展证明至关重要的设计决策。例如，我们的机器状态将内存建模为从字对齐地址到 32 位值的映射；仅推理对齐的内存访问简化了证明，因为对不同地址的访问是独立的。我们不直接建模虚拟内存翻译——加载和存储指令直接在操作数指定地址处操控内存映射的内容。这使我们可以仅基于有效地址值来定义地址有效性，而非基于整体机器状态。这加速了验证，因为证明器可以容易地看出有效性不受状态变化的影响。

Komodo 规约（第 5.2 节）确保监视器栈、全局变量和安全/非安全飞地页的有效地址区域。第 7.2 节后文描述了引导加载程序如何使用静态页表提供这些地址区域。

**用户模式执行。** 除了 ARM 用户模式提供的特权分离外，Komodo 的设计不约束可以在飞地中运行的代码。为建模飞地执行，我们可能因此需要建模每条允许的用户模式指令及其对机器状态的效果。这将意味着规约大量指令，以及更完整的机器状态和虚拟内存翻译模型。然而，我们不寻求验证在飞地中运行的代码，这样的模型将不必要地膨胀我们的可信计算基（TCB）并增加验证时间。

相反，我们仅建模推理 Komodo 正确性所需的用户模式执行方面，包括虚拟内存的有限视图：当用户代码执行时，它在发生异常之前会破坏（havoc）所有用户模式寄存器和所有用户可写页。可写页通过从页表基址寄存器开始遍历页表来找到，并翻译到监视器的内存映射中。本质上，我们规约监视器在某个固定虚拟偏移处以物理内存的 1:1 映射执行（由引导加载程序建立，见第 7.2 节）。我们通过使用偏移将可写页翻译到内存映射中来建模用户模式代码的效果。

作为惯用规约的另一个例子，ARM 支持多种页表格式，但我们只建模一种：短描述符格式的 4 kB" 小 " 页。如果遇到无法识别的页表条目，模型对用户执行结果不做任何断言——这迫使实现必须证明其页表满足规约，才能推理用户模式执行后的状态。

除页表外，我们还建模 TLB 一致性。执行 TLB 刷新指令将 TLB 标记为一致。加载页表基址寄存器，或对一级或任何二级页表中的地址执行存储，将 TLB 标记为不一致。这给实现留出了自由度：可以在需要一致性时简单地刷新 TLB，或者证明其存储操作没有修改页表。为简单起见，我们仅建模整个 TLB 的刷新（不包括基于标签或区域的刷新）。

**异常。** 我们处理异常的策略主要是避免它们。例如，加载和存储指令的前置条件排除了在经验证代码中发生页错误或对齐错误的可能性。然而，我们必须建模 CPU 中断使能标志（我们在第 7.2 节后文描述原因）。我们的指令求值核心规约声明：如果中断被使能，且如果（非确定性地）发生了中断，则指令仅在先运行中断处理程序之后执行，中断处理程序被建模为任意的实现特定谓词。这迫使正确的实现要么证明中断保持禁用，要么实现一个中断处理程序并证明在中断使能状态下执行的任何指令的前置条件被处理程序的后置条件所满足。

**局限性。** 我们的模型排除了实现使用许多与 Komodo 无关的特性的可能，包括 I/O 设备、大多数协处理器寄存器、浮点和向量状态，以及非对齐内存访问。由于我们不支持多核执行，我们不建模缓存或内存一致性。

### 5.2. Komodo 规约

除了 ARM 模型之外，我们还必须信任 Komodo 监视器的高层行为规约，同样用 Dafny 编写。我们通过证明该规约维护内部一致性不变量（下文描述）并保证高层安全属性（第 6 节描述）来增强信心。该规约的核心是 PageDB 的抽象表示：一个从页号到条目的映射，每个条目属于第 4 节前文描述的六种类型之一。

PageDB 表示抽象掉了与大部分规约无关的实现细节；例如：页表表示为抽象数据类型中的条目，为证明建立的飞地度量值表示为无界字序列。寄存器和内存的内容在飞地执行时当然很重要，因此规约包含一个谓词，描述飞地执行时可见的寄存器、内存和页表的内容。实现可以自由选择 PageDB 的内存表示，只要它能证明飞地执行时寄存器和虚拟内存的内容与抽象 PageDB 匹配即可。

我们规约的最高层是一个描述 SMC 处理程序的谓词。它将从 OS 发生 SMC 异常后的具体机器状态和抽象 PageDB 状态，关联到返回前的最终状态（s' 和 d'）：

```shell
predicate smchandler(s: state, d: PageDb,
                     s': state, d': PageDb)
```

在表 1 的所有监视器调用中，只有两个涉及飞地执行：`Enter` 和 `Resume`。我们将其余调用的主体规约为纯函数，给定输入 PageDB 和调用参数，计算错误/成功代码和结果 PageDB。顶层 `smchandler` 谓词在结果 PageDB 和错误代码与适当函数（基于调用号和参数寄存器）匹配时成立，且某些不变量在每次 SMC 中成立：非易失性寄存器被保留、其他非返回寄存器被清零（以防止信息泄露）、非安全内存不变，且以正确模式返回。

`Enter` 和 `Resume` 的规约也被建模为关联两个状态和 PageDB 的谓词。这些调用的规约迫使实现从高度约束的状态进入用户模式（它只能通过执行 `MOVS PC, LR` 来满足）。具体来说，页表基址寄存器必须加载飞地页表的地址，其内存中的表示必须与 PageDB 中编码的抽象页表匹配，TLB 必须一致。安全数据页的内容必须等于 PageDB 中的内容（飞地创建时的内容，或经飞地执行修改后的内容）。用户可见寄存器必须从 PageDB 加载：对于进入，PC 设为入口点，其他寄存器清零；对于恢复，用户可见寄存器从线程 PageDB 条目中保存的上下文恢复。

通过仅在进入用户模式时约束具体机器状态，我们保持了相当大的实现自由度。例如，实现可以任意格式维护其数据结构，只要它能证明用户模式执行环境满足规约。

来自飞地的 SVC 规约在逻辑上嵌套在 `Enter` 和 `Resume` 的定义内。用户模式执行后发生异常，规约随后根据异常状态确定调用结果和最终 PageDB。如果发生的异常是返回飞地的非 Exit SVC，则规约描述如何计算调用结果并返回执行飞地（使用递归定义的谓词）。所有其他异常更新 PageDB 并从 SMC 处理程序返回结果；例如，PageDB 的数据页被更新以反映飞地所做的任何更改；如果发生了中断，用户模式上下文必须保存在 PageDB 中且线程被标记为已进入。

有效的 PageDB 满足保证内部一致性的不变量：例如引用计数正确，内部引用（包括页表指针）指向属于同一地址空间的正确类型的页，且页表中映射的所有叶页要么是非安全页，要么是分配给同一地址空间的数据页。为增强对规约的信心，我们证明每个 SMC 和 SVC 保持 PageDB 不变量。这些不变量随后构成下一节安全证明的基础。

---

## 6. 安全证明

我们形式化证明上述 Komodo 规约保护飞地代码和数据的机密性和完整性免受机器上其他软件的侵害。由于实现被验证满足规约，这些安全证明也扩展到具体的 Komodo 代码。具体来说，我们证明飞地的内容不能被该飞地以外的任何软件修改，且飞地的内容不会泄露给其他飞地、操作系统或其他非飞地代码——除非飞地自身选择泄露，无论是直接的（例如写入非安全内存）还是间接的（例如通过飞地选择写入的非安全内存地址模式）。

更形式化地说，我们通过证明飞地与控制操作系统并与某飞地勾结的对手之间的无干扰性（noninterference）[32] 来建立 Komodo 的安全属性。除了有限的解密操作集（第 6.2 节）之外，我们分别建立机密性和完整性的两个独立结果：(i) 飞地状态与飞地外可观察状态之间无干扰；(ii) 飞地外软件可影响的状态与飞地状态之间无干扰。我们的飞地状态模型足以证明飞地数据和执行的机密性和完整性均得到保持。

从机密性角度看，无干扰性要求系统执行期间所有对手可观察的输出纯粹由对手提供的输入决定。换言之，公开输出永远不受秘密的影响。这是一个强端到端安全属性：它排除了秘密即使通过控制流间接影响公开输出的可能性。完整性是机密性的对偶 [10]，要求可信输出纯粹由可信输入决定。

通过建模一个同时控制操作系统和飞地的强对手，我们的结果推广到更简单的攻击者，如单独行动的操作系统或飞地。简而言之，我们形式化证明飞地秘密不会泄露给监视器以外的任何软件，且飞地不能被监视器以外的任何软件影响。

我们不寻求证明飞地正确使用 Komodo。飞地可能通过其返回代码、写入非安全内存或使用动态分配 SVC（一个我们在第 6.2 节中形式化描述的侧信道）直接泄露信息。它也可能通过硬件侧信道（例如缓存效应）间接泄露。如果飞地未能净化作为参数传递或从非安全内存读取的值，其完整性可能被破坏。与我们互补的工作为 SGX 飞地提供安全保证 [33, 80, 81]，可以适配 Komodo。

### 6.1. 规约

Komodo 在单核上执行，因此该核上的攻击者（包括潜在恶意飞地）不能与 Komodo 并发观察机器状态。然而，如第 5.1 节所述，我们确实允许操作系统在不同核上并发执行。操作系统不能观察寄存器或安全内存，但可以在 Komodo 执行期间并发访问非安全内存。我们的硬件模型通过禁止实现写入非安全内存（它没有理由这样做）来防止 Komodo 执行信息泄露到非安全内存。

我们利用攻击者只能在执行时进行观察这一事实来简化证明。我们只需推理系统中不同实体（正常世界、Komodo 监视器和飞地）之间的转换状态，因为对手不可能观察中间状态（例如非恶意飞地执行期间）。在这些转换点推理比推理完整执行轨迹更简单。

我们系统中的转换点在 SMC 和飞地执行的开始和结束处。我们为每个监视器调用证明无干扰性定理，且如我们讨论的，这足以保证飞地执行开始和结束时的安全性。通过仔细构造前置和后置条件，使我们对初始状态的假设也适用于最终状态，我们确保结果推广到无限的 SMC 序列。

我们以形式化的关系 ≈_L 描述某观察者 L 的观察能力——两个状态通过 ≈_L 关联，当且仅当这两个状态在 L 看来相同。≈_L 的定义取决于所考虑的观察者。对于机密性证明，观察者是一个模拟操作系统与飞地勾结的对手 adv。对于完整性证明，观察者是可信飞地 enc。

我们形式化无干扰性属性为：

**定理 6.1（无干扰性）。** 令 SMC 处理程序从状态 (s,d) 开始执行并在状态 (s',d') 返回，记为 smchandler(s,d,s',d')。则：

∀(s₁,d₁), (s₂,d₂), (s'₁,d'₁), (s'₂,d'₂):
(s₁,d₁) ≈_L (s₂,d₂) ∧ smchandler(s₁,d₁,s'₁,d'₁) ∧ smchandler(s₂,d₂,s'₂,d'₂)
⟹ (s'₁,d'₁) ≈_L (s'₂,d'₂)

我们证明此定理的一个放松版本，并在第 6.2 节讨论放松的方式。

### 6.2. 解密

飞地在正常执行期间向操作系统释放少量信息：结束飞地执行的异常或中断的类型、传递给 `Exit` 的返回值以及退出调用被执行的事实。使用动态内存分配的飞地还通过侧信道泄露，因为操作系统可以观察到哪些备用页已被飞地分配以及哪些数据页已被飞地释放。按照任何强制无干扰性的实际系统的惯例，我们依赖解密（declassification）来允许上述通信——否则信息流策略会将其排除。我们精确控制哪些信息被解密的工作最接近有界释放模型 [73]。

解密通过四个公理纳入我们的证明中，每个公理都有精确控制可使用它们的状态转换的前置条件。

### 6.3. 证明与非确定性

我们的证明使用互模拟（bisimulation）；我们推理从由 ≈_L 关联的初始状态开始的两次执行，证明目标是表明最终状态也由 ≈_L 关联。证明被结构化为关于每个监视器调用和每个 SVC 的更小互模拟证明。

`Enter` 和 `Resume` 的证明最为复杂，因为它们涉及飞地执行。我们的规约通过使用特定于更新状态的未解释函数来建模非确定性。每个函数至少接受两个输入：(i) 所有用户可见状态，包括通用寄存器、进入飞地时的 PC 以及当前页表可访问的所有内存；(ii) 一个建模为未知整数种子的非确定性源。对于两个无干扰性证明，我们要求观察者飞地的成功执行中初始状态的种子相同。这允许我们证明更新是确定性发生的。

机密性证明必须表明秘密飞地状态不会通过对手在监视器调用期间可观察的寄存器或非安全内存页泄露给对手。飞地被允许写入非安全内存。然而，正确的飞地代码不应将任何秘密写入非安全内存。为建模非安全内存是公开的这一事实，飞地对其的更新与安全内存不同处理：它们仍然是非确定性的，但不依赖于用户状态。

---

## 7. 实现

### 7.1. Vale 语言

图 2 显示了用于实现和验证 Komodo 的工具。我们的实现使用 Vale[12] 编程语言。Vale 程序由汇编语言指令和注解组成，如前置条件、后置条件和循环不变量，它们描述指令的行为。

Vale 工具生成两个中间 Dafny 语言对象：指令的抽象语法树（AST）表示，以及关于指令行为的声称证明（例如指令确保注解中的后置条件）。如果注解或代码有误，此证明将无效。

由于 Komodo 由低级汇编代码组成，我们不大量依赖 Dafny 的可执行代码功能。实际上，唯一的可执行 Dafny 代码是一个简单的美观打印器，将指令 AST 转换为 GNU 汇编格式。此打印器，连同 Dafny 和可信规约，构成 Komodo 验证的可信计算基。Vale 工具不是可信计算基的一部分；Vale 中的缺陷可能创建不正确的 AST 或无效的证明，Dafny 会因未正确满足规约而拒绝此类 AST 或证明。

### 7.2. 实现细节

**硬件平台。** 我们的原型运行在 Raspberry Pi 2 上，它广泛可用且包含支持 TrustZone 的 CPU 和硬件随机数生成器，但缺乏对隔离安全世界内存或执行硬件支持证明的支持。相反，我们简单地假设存在一个静态配置的隔离内存区域和硬件派生的证明秘密，并依赖引导加载程序来提供它们。这意味着我们的原型不幸不提供实际安全性；然而，将其移植到包含这些功能的 ARM 平台上既不会改变其性能也不会改变证明，因为两个功能仅影响启动时配置。

**异常处理程序。** 实现面临的最大挑战之一是 Vale 建模的线性控制流（它自动化了具有单一起始和结束状态的过程验证）与内核代码固有的异常驱动执行风格之间的不匹配。Komodo 没有单个顶层过程；相反，它由从硬件异常向量表调用的处理程序实现。这些包括由 OS 和飞地分别调用的 SMC 和 SVC 处理程序、ARM 两种不同中断（"FIQ" 和 "IRQ"）的处理程序，以及未定义指令和数据中止（页错误）的异常处理。

如图 3 所示，这些处理程序形成一个状态机，嵌套在顶层 SMC 处理程序内部并受其规约约束。我们在 Dafny 中显式建模状态转换，证明每当异常可能发生时，其前置条件被满足且其处理程序建立下一状态所需的条件。

**中断。** 只要可能，监视器在中断禁用状态下执行。这允许我们孤立地推理大多数指令，这是一个合理的权衡，因为所有操作都是有界时间的（运行时间最长的监视器调用 `MapSecure` 初始化并哈希单页内存）。在执行飞地时中断被使能，在发生 SMC 或中断异常时自动禁用。然而，在进入系统调用、中止或未定义指令的处理程序之后可能发生中断。我们的异常处理程序立即禁用中断，但存在不可避免的单指令窗口，在此期间嵌套异常可能发生。

中断处理程序的行为取决于系统的先前状态。如果中断在用户模式下发生，它定位当前线程页并在分支到后续代码之前保存用户模式寄存器上下文。然而，如果中断在特权模式下发生，它仅设置一个标志记录中断发生，然后恢复执行，寄存器和内存保持不变且中断被禁用。

**飞地执行。** `Enter` 和 `Resume` 的实现必须执行飞地无限次，直到发生 Exit SVC 或异常。在我们的线性控制流模型中，这需要以下形式的循环：

```c
while (!done) {
    MOVS_PC_LR(); // 进入用户模式，
                   // 处理异常，分支回来
}
```

然而，由于在 done 测试点每个用户可见寄存器都是活跃的（甚至测试全局变量也需要一个备用寄存器），我们被迫使用监视器 SP 寄存器的最低有效位作为 done 标志。我们的见解是，用丑陋（但经过验证的）代码污染部分实现，比增加执行模型（从而增加 TCB）的复杂性更为可取。

**内存映射。** 图 4 显示了安全世界虚拟内存布局。我们利用 ARM 架构特性将监视器的页表与飞地地址空间使用的页表解耦——飞地的页表加载到 TTBR0 控制寄存器中，该寄存器被配置为仅映射虚拟地址空间的前 1GB（飞地的上限地址）。剩余地址空间由引导加载程序创建的 TTBR1 中的单独静态页表映射。后者区域限制为特权模式，包括监视器代码和数据（栈和全局变量）的映射，以及物理内存的大型直接映射。后者又包括引导加载程序分配的隔离内存，飞地就存在于此。

**证明与密码学。** Komodo 借用了先前 Vale 工作 [12] 中的核心 ARM SHA-256 实现。因此，我们受益于良好的哈希性能（因为代码与 OpenSSL 中优化的 SHA 例程相对应），以及无数字（缓存和时序）侧信道的证明。我们还实现了基于 SHA-256 的 MAC，用于生成和验证证明。

### 7.3. 代码规模与验证工作量

**表 2 — 代码行数**

| 组件 | 规约 | 实现 | 证明 | 汇编指令数 |
|------|------|------|------|-----------|
| ARM 模型 | 1,174 | 112 | 985 | — |
| Dafny 库 | 588 | — | 806 | — |
| SHA-256, SHA-HMAC | 250 | 415 | 3,200 | 170 |
| Komodo 公共部分 | 775 | 358 | 3,078 | 136 |
| SMC 处理程序 | 591 | 1,082 | 4,493 | 284 |
| SVC 处理程序 | 204 | 612 | 2,509 | 233 |
| 其他异常 | 39 | 131 | 940 | 52 |
| 无干扰性 | 175 | — | 2,644 | — |
| 汇编打印器 | 650 | — | — | — |
| **总计** | **4,446** | **2,710** | **18,655** | **875** |

规约行数包括所有可信 Dafny 代码。实现行数是我们在 Vale 中编写的汇编指令、过程调用和控制流。证明行数是帮助验证器的注解。打印后，经过验证的原型被输出为 26,800 行汇编文件。

Komodo 的完整端到端验证需要 4 个核心小时。然而它高度并行化，支持分布式验证。典型开发者一次处理一个过程或引理，大多数验证耗时远不到一分钟。

排除继承自先前工作 [12] 的核心 SHA 实现，我们总共花费约 2 个人年来规约和实现 Komodo。我们从 ARM 模型的规约开始，然后规约并实现了模仿 SGXv1 的简化版 Komodo（使用静态内存管理）。构建第一个版本花费了约 1.5 个人年，包括主要开发者不熟悉验证工具（当时正在积极开发中）的陡峭学习曲线。

在开发第一个版本的过程中，我们经历了若干阶段，在这些阶段中我们重构了核心定义（例如 ARM 机器模型），使其更适合自动化验证，并逐步建模更复杂的特性（如异常和中断）。每次此类迭代都需要修订现有证明以维护新的不变量。

例如，我们发现关于字对齐的推理对 Z3 来说代价过高，间接导致验证超时。因此我们更改了字对齐地址的核心定义为不透明函数，并证明了关于它的选定引理。这需要更改所有执行内存访问或操控地址的过程以使用新的声明和引理，导致了一周的半机械式重构，但显著改善了证明稳定性。

我们随后用动态内存管理扩展了规约和实现；这额外花费了 6 个人月，包括用于无干扰性证明更新的 3 个人周。鉴于这些改进，我们估计以稳定的工具和规约重新构建第一个版本所需的时间将远少于 1 年。

---

## 8. 评估

### 8.1. 微基准测试

我们在 Raspberry Pi 2 Model B（900MHz ARM Cortex-A7 CPU）上测试了原型。为此，我们实现了一个简单的引导加载程序，在安全世界中加载监视器，设置其内存映射和异常向量。引导加载程序还预留可配置量的 RAM 作为安全内存，然后切换到正常世界启动 Linux。一旦 Linux 启动，内核驱动程序发出 SMC 来创建和运行飞地。

**表 3 — Raspberry Pi 上的微基准测试结果**

| 操作 | 说明 | 周期数 |
|------|------|--------|
| GetPhysPages | 空 SMC | 123 |
| Enter + Exit | 完整飞地穿越（调用与返回） | 738 |
| Enter only | 仅进入（无返回） | 496 |
| Resume only | 仅恢复（无返回） | 625 |
| Attest | 构造证明 | 12,411 |
| Verify | 验证证明 | 13,373 |
| AllocSpare | 动态分配 | 217 |
| MapData | 动态分配 | 5,826 |

原型监视器完全未优化。它保守地保存和恢复每个非易失性寄存器——对于 `GetPhysPages` 这样的简单 SMC 来说是不必要的开销。在飞地进入时，它还保存和恢复每个分组寄存器（尽管有些已知被保持不变），并刷新 TLB（尽管对同一飞地的重复调用可以避免这样做）。这些都是我们计划添加的优化——但只有在证明其正确性之后。

尽管缺乏优化，Komodo 的性能与 SGX 相比表现良好。Orenbach 等人 [66, §2.2] 报告 EENTER 和 EEXIT 延迟分别约为 3,800 和 3,300 个周期，即一次完整飞地穿越为 7,100 个周期。当然，x86 运行在更高的时钟频率（2GHz vs. 900MHz）且包含内存加密，但 Komodo 的结果代表了一个数量级的改进。我们只能推测原因，但显然用软件实现飞地没有固有的性能惩罚。

### 8.2. 公证人飞地

为了用真实飞地测试 Komodo 并帮助确信其 API 的完整性，我们移植了 Ironclad[38, §5.1] 中的可信公证人应用。公证人为文档分配逻辑时间戳，使其可以被确定性地排序。我们将公证人重新实现为针对 Komodo 飞地 API 编译的 3700 行独立 C 程序。首次进入时，它构造 RSA 密钥对，初始化单调计数器，并构造和返回其初始状态的证明。在后续调用中，它使用当前计数器值对提供的文档进行哈希并用 RSA 密钥签名，然后递增计数器并返回签名。公证人的总内存占用为 145 kB。性能测量（图 5）表明，由于其执行由 CPU 密集的哈希和签名操作主导，公证人在飞地中的性能与原生 Linux 进程等同。

---

## 9. 讨论

### 9.1. 经验教训

**小代码库不能替代验证。** 在着手验证 Komodo 之前，我们曾先用 C 和汇编实现了一个未经验证的版本，以熟悉 TrustZone 设计。这个未经验证的监视器仅约 650 行 C 和 300 行汇编（不支持证明），但它仍然包含在规约和实现 Komodo 过程中才暴露出来的关键安全缺陷。

例如，`InitAddrspace` 接受两个页号。未经验证的实现检查两者都空闲，然后继续分配并初始化地址空间。只有在编写此调用的规约并未能证明其维护 PageDB 不变量后，我们才发现没有考虑两个参数为同一页的情况。

作为一个更微妙的问题，在检查非安全内存页的有效性时，我们未能考虑到监视器的代码和数据同时存在于直接映射的物理内存和虚拟内存中（见图 4）。要检查传递给监视器的非安全物理地址是否有效，仅检查它不指向安全页是不够的；它还必须避开监视器自身的任何页。

**可信组件需要额外的谨慎。** 在验证任何系统时，都必须选择信任什么和验证什么。我们在运行代码时发现了缺陷；不出所料，它们都在可信代码或我们系统的规约不足的部分 [28] 中。例如：一个汇编打印器中的缺陷导致所有意图操作分组 SPSR 寄存器的指令改为使用当前模式的 SPSR；我们在访问某些控制寄存器时缺少屏障（DSB 和 ISB 指令）；引导加载程序、监视器和 Linux 驱动程序之间缓存和页属性配置的不一致导致了正常世界和安全世界共享页视图的缓存不一致。

我们的结论是：虽然验证在消除整类错误方面具有巨大价值，但它不能阻止开发者做出任何不当假设——至少在没有 CPU 行为的完整正确形式化规约的情况下。

**改进验证工具的机会仍然存在。** 最令人沮丧的反复出现的问题是证明不稳定性。对于简单引理，Dafny 要么报告成功，要么报告具体失败。然而，随着证明复杂性增加，求解器时间可能呈指数增长。在 Komodo 中，只要推理具有许多指令（因而许多状态转换）或复杂规约的过程，这就容易发生。为避免无尽搜索，Dafny 实现了超时限制。超时难以调试，因为求解器通常无法提供有用的反馈。唯一可靠地消除超时的方法通常是将代码重构为具有自身显式前置和后置条件的更小子过程，但这导致不优雅且难以维护的代码。

### 9.2. 未来工作

**调度器接口。** Komodo 之所以不易受受控信道攻击 [78, 88]，仅仅是因为它尚不支持飞地内存的按需分页。我们希望将当前基于线程的接口演进为 LibOS 风格的调度器接口 [55]，具有显式的用户模式向上调用以恢复线程或报告异常。这将允许使用飞地自分页来管理内存 [37, 66]，而不向不可信操作系统暴露页错误。

**多核支持。** Komodo 目前最大的局限无疑是多核支持。有几种途径可以弥合这一差距，但最简单的是在所有监视器活动周围使用单个共享锁，这将保留我们当前证明中使用的顺序（Floyd-Hoare）推理。微内核的经验甚至表明这可能不会过度损害性能 [25]。

---

## 10. 相关工作

**硬件。** 大量系统使用硬件将敏感代码与不可信操作系统隔离 [15, 16, 26, 27, 50, 53, 67, 82]。它们在抵御硬件攻击的能力、软件可信计算基的大小和保护粒度方面各不相同。然而，据我们所知，没有一个具有形式化规约或安全证明。SGX[43, 59] 的独特之处主要在于它在 x86 中的实现。

与 Komodo 最密切相关的系统是 Sanctum[19]。与 Komodo 一样，Sanctum 由简单的硬件扩展组成，以支持可信安全监视器来管理和保护飞地。与 Sanctum 不同的是，Komodo 原型运行在现成的硬件（ARM TrustZone）上，并包含功能正确性和保证飞地完整性与机密性的无干扰性属性的机器可检查证明。

Sanctum 和 Komodo 在证明方式上也有所不同。Sanctum 在监视器中计算度量值，但将证明委托给特权签名飞地以避免涉及证明密钥的侧信道泄露。我们改为在监视器中直接实现本地证明。我们的证明算法（HMAC-SHA256）在其地址轨迹中是数据无关的，我们可以使用先前为 Vale 开发的技术 [12] 来证明这一点。

**软件。** 其他系统也尝试使用通用硬件在软件中提供类飞地的隔离执行环境。然而，其中大多数未提供形式化保证 [例如 57, 58, 69]。

我们的验证方法论建立在 Verve[89]、Ironclad[38] 和 IronFleet[39] 中开发的工具和技术之上，然而最密切相关的经过验证的系统是内核 seL4[49]、CertiKOS[34, 35] 和 überSpark[84]。Komodo 甚至可以被视为具有不寻常 API 的微内核，但这种比较有其局限性：Komodo 不处理中断也不支持设备驱动程序，甚至不包括系统时钟或中断控制器；它不实现调度器也不执行资源管理；它缺少 IPC 机制。但它确实执行证明并与不可信操作系统并行运行。与 Komodo 一样，seL4 和 CertiKOS 也受益于基于无干扰性的安全属性证明 [20, 62]；seL4 的形式化更为复杂，因为它包含通用 IPC。

最终，Komodo 从简洁性和自动化验证工具中获得的优势是适应性：我们可以快速演进 Komodo 同时保持其安全保证，而复杂内核如 seL4 和 CertiKOS 则代表着相当多的人力投入。

---

## 11. 结论

Komodo 是类 SGX 飞地隔离机制的首个经过形式化验证的实现。其设计将飞地硬件原语与安全关键但经过形式化验证的软件解耦，使两者能够独立演进。我们使用无干扰性证明了机密性和完整性的高层保证，证明了该方法是可行的，Komodo 可以比 SGX 更快地演进，甚至在性能上超越 SGX。

---

## 致谢

我们感谢匿名审稿人和我们的指导委员 George Candea 提供的宝贵反馈，以及 Rustan Leino 和 Jay Lorch 在 Dafny 方面的帮助。

本工作部分由美国国家科学基金会和 VMware 资助，资助编号 CNS-1700521。

---

## 参考文献

[1] PrimeCell Infrastructure AMBA 3 TrustZone Protection Controller (BP147) Technical Overview. ARM Limited, Nov. 2004.

[2] Building a Secure System using TrustZone Technology. ARM Limited, Apr. 2009.

[3] ARM Architecture Reference Manual, ARMv7-A and ARMv7-R edition. ARM Limited, May 2014.

[4] ARM CoreLink TZC-400 TrustZone Address Space Controller Technical Reference Manual. ARM Limited, Feb. 2014.

[5] S. Arnautov et al. SCONE: Secure Linux containers with Intel SGX. In *OSDI*, 2016.

[6] A. M. Azab et al. Hypervision across worlds: Real-time kernel protection from the ARM TrustZone secure world. In *ACM CCS*, 2014.

[7] A. Baumann. Hardware is the new software. In *HotOS '17*, 2017.

[8] A. Baumann, M. Peinado, and G. Hunt. Shielding applications from an untrusted cloud with Haven. In *OSDI*, 2014.

[9] J. Behl, T. Distler, and R. Kapitza. Hybrids on steroids: SGX-based high performance BFT. In *EuroSys*, 2017.

[10] K. J. Biba. Integrity considerations for secure computer systems. Technical Report, USAF, 1977.

[11] R. Boivie. SecureBlue++: CPU support for secure execution. Technical Report, IBM Research, 2012.

[12] B. Bond et al. Vale: Verifying high-performance cryptographic assembly code. In *USENIX Security*, 2017.

[13] F. Brasser et al. Software grand exposure: SGX cache attacks are practical. In *WOOT '17*, 2017.

[14] S. Brenner et al. SecureKeeper: Confidential ZooKeeper using Intel SGX. In *Middleware*, 2016.

[15] D. Champagne and R. B. Lee. Scalable architectural support for trusted software. In *HPCA*, 2010.

[16] S. Chhabra et al. SecureME: a hardware-software approach to full system security. In *ICS*, 2011.

[17] P. Colp et al. Protecting data on smartphones and tablets from memory attacks. In *ASPLOS*, 2015.

[18] V. Costan and S. Devadas. Intel SGX explained. Cryptology ePrint Archive, 2016.

[19] V. Costan, I. Lebedev, and S. Devadas. Sanctum: Minimal hardware extensions for strong software isolation. In *USENIX Security*, 2016.

[20] D. Costanzo, Z. Shao, and R. Gu. End-to-end verification of information-flow security for C and assembly programs. In *PLDI*, 2016.

[21] S. Crosby. Using Intel SGX to protect on-line credentials. 2016.

[22] I. Cutress. Intel's 'Tick-Tock' seemingly dead. *AnandTech*, 2016.

[23] L. de Moura and N. Bjørner. Z3: An efficient SMT solver. In *TACAS*, 2008.

[24] PALcode for Alpha Microprocessors System Design Guide. Digital Equipment Corp., 1996.

[25] K. Elphinstone et al. An evaluation of coarse-grained locking for multicore microkernels. *CoRR*, 2016.

[26] D. Evtyushkin et al. Iso-X: A flexible architecture for hardware-managed isolated execution. In *MICRO-47*, 2014.

[27] C. W. Fletcher, M. v. Dijk, and S. Devadas. A secure processor architecture for encrypted computation on untrusted programs. In *STC*, 2012.

[28] P. Fonseca et al. An empirical study on the correctness of formally verified distributed systems. In *EuroSys*, 2017.

[29] A. Fox and M. O. Myreen. A trustworthy monadic formalization of the ARMv7 instruction set architecture. In *ITP*, 2010.

[30] GlobalPlatform Device Technology TEE System Architecture v1.1. GlobalPlatform, 2017.

[31] A. Goel et al. SMT-based system verification with DVF. In *SMT Workshop*, 2012.

[32] J. A. Goguen and J. Meseguer. Security policies and security models. In *IEEE S&P*, 1982.

[33] A. Gollamudi and S. Chong. Automatic enforcement of expressive security policies using enclaves. In *OOPSLA*, 2016.

[34] R. Gu et al. Deep specifications and certified abstraction layers. In *POPL*, 2015.

[35] R. Gu et al. CertiKOS: An extensible architecture for building certified concurrent OS kernels. In *OSDI*, 2016.

[36] J. A. Halderman et al. Lest we remember: Cold boot attacks on encryption keys. In *USENIX Security*, 2008.

[37] S. M. Hand. Self-paging in the Nemesis operating system. In *OSDI*, 1999.

[38] C. Hawblitzel et al. Ironclad apps: End-to-end security via automated full-system verification. In *OSDI*, 2014.

[39] C. Hawblitzel et al. IronFleet: Proving practical distributed systems correct. In *SOSP*, 2015.

[40] J. Howell, B. Parno, and J. R. Douceur. Embassies: Radically refactoring the web. In *NSDI*, 2013.

[41] G. Hunt, G. Letey, and E. Nightingale. The seven properties of highly secure devices. Technical Report, Microsoft Research, 2017.

[42] T. Hunt et al. Ryoan: A distributed sandbox for untrusted computation on secret data. In *OSDI*, 2016.

[43] Software Guard Extensions Programming Reference. Intel Corp., Oct. 2014.

[44] SGX Tutorial at ISCA 2015. Intel Corp., June 2015.

[45] Intel 64 and IA-32 Architectures Software Developer's Manual. Intel Corp., Dec. 2016.

[46] S. P. Johnson et al. Technique for supporting multiple secure enclaves. US Patent 8,972,746, 2010.

[47] U. Kanonov and A. Wool. Secure containers in Android: The Samsung KNOX case study. In *SPSM*, 2016.

[48] D. Kaplan, J. Powell, and T. Woller. AMD memory encryption. AMD Whitepaper, 2016.

[49] G. Klein et al. Comprehensive formal verification of an OS microkernel. *ACM TOCS*, 32(1), 2014.

[50] R. B. Lee et al. Architecture for protecting critical secrets in microprocessors. In *ISCA*, 2005.

[51] K. R. M. Leino. Dafny: An automatic program verifier for functional correctness. In *LPAR-16*, 2010.

[52] R. Leslie-Hurd, D. Caspi, and M. Fernandez. Verifying linearizability of Intel software guard extensions. In *CAV*, 2015.

[53] D. Lie et al. Architectural support for copy and tamper resistant software. In *ASPLOS*, 2000.

[54] L. Maranget, S. Sarkar, and P. Sewell. A tutorial introduction to the ARM and POWER relaxed memory models. Draft, 2012.

[55] B. D. Marsh et al. First-class user-level threads. In *SOSP*, 1991.

[56] J. M. McCune et al. Minimal TCB code execution. In *IEEE S&P*, 2007.

[57] J. M. McCune et al. Flicker: an execution infrastructure for TCB minimization. In *EuroSys*, 2008.

[58] J. M. McCune et al. TrustVisor: Efficient TCB reduction and attestation. In *IEEE S&P*, 2010.

[59] F. McKeen et al. Innovative instructions and software model for isolated execution. In *HASP*, 2013.

[60] F. X. McKeen et al. Method and apparatus to provide secure application execution. US Patent 9,087,200, 2009.

[61] MITRE. CVE-2017-5691, July 2017.

[62] T. Murray et al. seL4: From general purpose to a proof of information flow enforcement. In *IEEE S&P*, 2013.

[63] K. T. Nguyen. Introduction to Cache Allocation Technology in the Intel Xeon Processor E5 v4 family. Intel, 2016.

[64] NXP. i.MX 7Solo, i.MX 7Dual applications processors, 2017.

[65] O. Ohrimenko et al. Oblivious multi-party machine learning on trusted processors. In *USENIX Security*, 2016.

[66] M. Orenbach et al. Eleos: ExitLess OS services for SGX enclaves. In *EuroSys*, 2017.

[67] E. Owusu et al. OASIS: On achieving a sanctuary for integrity and secrecy on untrusted platforms. In *ACM CCS*, 2013.

[68] R. Pires et al. Secure content-based routing using Intel software guard extensions. In *Middleware*, 2016.

[69] H. Raj et al. Credo: Trusted computing for guest VMs with a commodity hypervisor. Technical Report, Microsoft Research, 2011.

[70] H. Raj et al. fTPM: A software-only implementation of a TPM chip. In *USENIX Security*, 2016.

[71] A. Reid. Trustworthy specifications of ARM v8-A and v8-M system level architecture. In *FMCAD*, 2016.

[72] A. Reid et al. End-to-end verification of ARM processors with ISA-formal. In *CAV*, 2016.

[73] A. Sabelfeld and A. C. Myers. A Model for Delimited Information Release. Springer, 2004.

[74] N. Santos et al. Using ARM TrustZone to build a trusted language runtime for mobile applications. In *ASPLOS*, 2014.

[75] F. Schuster et al. VC3: Trustworthy data analytics in the cloud using SGX. In *IEEE S&P*, 2015.

[76] M. Schwarz et al. Malware guard extension: Using SGX to conceal cache attacks. In *DIMVA*, 2017.

[77] M.-W. Shih et al. T-SGX: Eradicating controlled-channel attacks against enclave programs. In *NDSS*, 2017.

[78] S. Shinde et al. Preventing page faults from telling your secrets. In *ACM ASIACCS*, 2016.

[79] T. Simonite. Intel puts the brakes on Moore's Law. *MIT Technology Review*, 2016.

[80] R. Sinha et al. Moat: Verifying confidentiality of enclave programs. In *ACM CCS*, 2015.

[81] R. Sinha et al. A design and verification methodology for secure isolated regions. In *PLDI*, 2016.

[82] J. Szefer and R. B. Lee. Architectural support for hypervisor-secure virtualization. In *ASPLOS*, 2012.

[83] TPM Main Specification Level 2. Trusted Computing Group, 2011.

[84] A. Vasudevan et al. überSpark: Enforcing verifiable object abstractions for automated compositional security analysis of a hypervisor. In *USENIX Security*, 2016.

[85] A. Waterman et al. The RISC-V instruction set manual volume II: Privileged architecture version 1.7. Technical Report, UC Berkeley, 2015.

[86] R. Wojtczuk and J. Rutkowska. Attacking Intel TXT via SINIT code execution hijacking. 2011.

[87] R. Wojtczuk, J. Rutkowska, and A. Tereshkin. Another way to circumvent Intel Trusted Execution Technology. 2009.

[88] Y. Xu, W. Cui, and M. Peinado. Controlled-channel attacks: Deterministic side-channels for untrusted operating systems. In *IEEE S&P*, 2015.

[89] J. Yang and C. Hawblitzel. Safe to the last instruction: Automated verification of a type-safe operating system. In *PLDI*, 2010.

<!-- obsidian-publish-source: Operating System 操作系统/Pappers 论文/SOSP/Komodo 利用验证将安全飞地硬件与软件解耦.md -->