## EuroSys 2026 论文综述

**会议:** 第 21 届欧洲计算机系统会议 (EuroSys '26)
**地点:** 英国爱丁堡，2026 年 4 月 27–30 日
**论文集:** ACM (DOI: 10.1145/3767295)

EuroSys 2026 共录用约 90 余篇论文，覆盖系统研究的完整栈。以下按主要研究方向分类综述，梳理关键趋势与代表性工作。

---

### 一、LLM 推理与服务（主导方向 — 约 15 篇）

这是 EuroSys 2026 中规模最大的论文簇，反映了工业界对高效 LLM 服务的迫切需求。

| 论文 | 核心思路 |
|------|----------|
| **AdaServe** | 基于 SLO 定制的投机解码，加速多 SLO LLM 服务 (CMU) |
| **FlexPipe** | 在碎片化 Serverless 集群中通过运行时流水线重构实现动态 LLM 服务 (中科院/UCSD) |
| **TokenFlow** | 请求突发下通过抢占式调度实现响应式文本流式服务 (上交) |
| **KunServe** | 面向参数的内存管理，高效处理 LLM 服务中的内存过载 (上交) |
| **AdaGen** | 负载自适应的集群调度器，优化推理延迟 (UVA/HPE) |
| **SkyWalker** | 面向跨地域 LLM 推理的局部性感知负载均衡器 (UC Berkeley) |
| **MFS** | 模型家族服务系统——跨相关 LLM 共享参数 (港科大) |
| **LLMFolder** | 将编译器常量折叠优化应用于 LLM 推理 (上交) |
| **SAS** | 稀疏注意力合成器，提升推理效率 (Amazon) |
| **TailorLLM** | 基于 LoRA 的端云协同推理 (北邮) |
| **AIMS** | 混合云 - 边环境下的低成本 LLM Agent 部署 (UVA/Microsoft) |
| **Taming MoE** | 细粒度专家卸载，平衡延迟与内存 (Stevens/Rice/Waterloo) |
| **TZ-LLM** | 利用 ARM TrustZone 保护端侧 LLM (上交) |
| **Scaling LLM Test-Time Compute** | 在手机 NPU 上扩展 LLM 测试时计算 (清华/Microsoft) |

**趋势:** 社区正从追求原始吞吐转向差异化 SLO 保障、内存高效服务和边缘/移动端部署。跨地域推理和多模型共服务是新兴方向。

---

### 二、大规模模型训练（约 12 篇）

| 论文 | 核心思路 |
|------|----------|
| **MegaScale-MoE** | 生产级通信高效 MoE 训练 (字节跳动/北大) |
| **LoRAFusion** | 高效 LoRA 微调系统 (多伦多大学/NVIDIA) |
| **STAlloc** | 时空规划的训练内存优化 (清华/无穹) |
| **Zeppelin** | 数据并行中变长负载均衡 (北大/ETH) |
| **Efficient Overlapping** | 通过信号和重排实现计算 - 通信重叠 (清华) |
| **HARP** | 异构 GPU 集群自动并行训练编排 (复旦) |
| **Crimson** | 流水线并行中的协作参数更新 (中山大学) |
| **Suika** | 共享集群中 3D 并行作业的高质量重调度 (上交/TeleAI) |
| **Handling Network Faults** | 分布式 AI 训练中的网络故障容错 (NUS/字节跳动) |
| **Maya** | 基于 GPU 运行时仿真的训练优化 (Georgia Tech) |
| **MinatoLoader** | 高效数据预处理流水线 (McGill) |
| **SwiftFL** | 端侧联邦学习的投机训练 (中科院) |
| **Federated Fine-Tuning** | 资源受限设备上的稀疏 MoE 联邦微调 (山大/西交) |

**趋势:** 生产级 MoE 大规模训练、多节点训练的容错能力、异构集群调度是核心主题。计算与通信的重叠仍是被积极攻关的基础难题。

---

### 三、网络与通信（约 10 篇）

| 论文 | 核心思路 |
|------|----------|
| **Learn-to-Probe** | 基于学习的拥塞控制中实现信号可区分性 (港科大) |
| **REPS** | 回收熵包喷洒的自适应负载均衡与故障缓解 (ETH/Microsoft) |
| **Multipath Collective** | GPU 云中超越 Scale-up 网络的多路径集合通信 (北大/腾讯) |
| **Concord** | 学习网络配置契约 (Microsoft Research) |
| **Canopy** | 属性驱动的学习型拥塞控制 (UT Austin) |
| **PatternSketch** | 运行时可重配置的网络流量模式检测 (苏大) |
| **Rearchitecting Programmable Networks** | 面向网内计算的可编程网络重构 (北大/华为) |
| **Practical RDMA Sharing** | 面向 HPC 的可扩展 RDMA 连接共享 (北大/华为) |
| **RLive** | 大规模直播流的鲁棒传输系统 (中科院计算所/字节跳动) |
| **NutCracker** | 混合 DPU 架构的编译框架 (NUS/MPI-SWS) |

**趋势:** 学习型拥塞控制和网络管理日趋成熟。受 AI 训练需求驱动，GPU 云通信基础设施（集合通信、RDMA、DPU）获得了大量关注。

---

### 四、Serverless 计算（约 7 篇）

| 论文 | 核心思路 |
|------|----------|
| **iRoute** | 基于本地路由表的 Serverless 工作流管理 (天津大学) |
| **DROPS** | Azure Functions 规模的 Serverless 资源池管理 (Waterloo/Microsoft) |
| **Squeezy** | 面向 Serverless 函数的快速 VM 内存回收 (雅典工大) |
| **Demystifying Serverless Costs** | 桥接计费、架构与 OS 调度的成本分析 (UBC) |
| **Efficient Data Passing** | 面向 Serverless 推理工作流的 GPU 中心数据传递 (华科) |
| **Serverless Replication** | 跨多厂商云的对象存储无服务器复制 (北大) |
| **In-Production Characterization** | 开源 Serverless 平台生产环境画像及新扩缩策略 (UBC/IBM) |

**趋势:** Serverless 正从 FaaS 向*AI 推理服务*演进。冷启动和资源管理仍然活跃，但成本建模和 GPU 集成是新前沿。

---

### 五、操作系统、虚拟化与容器（约 8 篇）

| 论文 | 核心思路 |
|------|----------|
| **CofferOS** | 基于 Rust 加固的 OS 级虚拟化（隔离性 + 可定制性） |
| **Pyramid** | 安全、资源高效的多租户 Kubernetes (清华/蚂蚁) |
| **SKernel** | 分裂内核架构——大规模弹性安全容器 (清华/蚂蚁/上交) |
| **NecoFuzz** | 通过 Fuzz-Harness 虚拟机模糊测试嵌套虚拟化 (东京大学) |
| **PaCaR** | NUMA 系统上通过 Page Cache 复制改善缓冲 I/O 局部性 (亚琛工大) |
| **VM Live Migration** | 异构处理器间的虚拟机热迁移 (格勒诺布尔大学) |
| **Chimera** | 通过二进制重写实现透明异构计算 (中科院软件所) |
| **Yield Not Thy Core** | 重新审视系统中的核心让渡机制 (UC Santa Cruz) |

**趋势:** 大规模容器安全（分裂内核、Rust 加固隔离）和异构处理器虚拟化是突出方向。社区还在将*模糊测试*虚拟化层作为一等公民测试方法。

---

### 六、安全、隐私与模糊测试（约 6 篇）

| 论文 | 核心思路 |
|------|----------|
| **TZ-LLM** | ARM TrustZone 保护端侧 LLM (上交) |
| **Turnstile** | 面向 IoT 隐私的混合信息流控制框架 (UBC/Princeton) |
| **Five Minutes of DDoS Brings down Tor** | 针对 Tor 目录协议的 DDoS 攻击及防御 (Purdue) |
| **LifeFuzz** | 生命周期引导的 Windows 驱动跨处理函数漏洞模糊测试 (中科院) |
| **Effective On-Hardware Fuzzing** | 在真实硬件上模糊测试嵌入式 OS (清华) |
| **TAO** | 面向浮点神经网络的容差感知乐观验证 (Princeton) |

**趋势:** 机密计算（TrustZone 保护 LLM）、协议级攻击（Tor）和*系统级模糊测试*（驱动、嵌套虚拟化、嵌入式 OS）是活跃领域。安全研究正从传统 OS 加固扩展到 AI 模型保护。

---

### 七、存储与文件系统（约 5 篇）

| 论文 | 核心思路 |
|------|----------|
| **SwitchFS** | 基于网内协调的分布式文件系统异步元数据更新 (上交) |
| **Omar** | 云块存储的主动 + 被动混合调度 (上交/阿里巴巴) |
| **PASS** | 功率自适应存储服务器 (华盛顿大学) |
| **Fast Crash Consistency** | 机会性顺序消除实现并行化崩溃一致性 (哈工大深圳) |
| **FicusDB** | 可扩展的多版本认证归档存储 (Cornell) |

**趋势:** 元数据的网内协调、功率感知存储和崩溃一致性依然重要。阿里巴巴生产规模的云块存储调度 (Omar) 体现了产学差距的缩小。

---

### 八、内存系统与硬件加速（约 7 篇）

| 论文 | 核心思路 |
|------|----------|
| **TierScape** | 多级压缩内存层降低服务器 TCO (Intel Labs) |
| **FUR** | 持久内存事务上的快速无限读 (INESC-ID) |
| **MTTM** | 多租户云的动态快速内存分区与带宽优化 (KAIST) |
| **BASK** | SmartNIC 卸载的 KSM（内核同页合并）(KAIST) |
| **LightDSA** | 面向 Data Streaming Accelerator 的硬件感知透明优化 (人大/阿里巴巴) |
| **Accelerating Transactions via PIM** | 基于存内计算加速事务执行 (INESC-ID) |
| **RoPeerTo** | 数据中心级 GPU 与 FPGA 间的点对点 DMA (Polimi/ETH/AMD) |

**趋势:** 分层内存（CXL/压缩）、SmartNIC 卸载（KSM）和异构 DMA 架构反映了解耦数据中心的发展方向。Intel DSA 和存内计算正获得系统级关注。

---

### 九、分布式系统与共识（约 6 篇）

| 论文 | 核心思路 |
|------|----------|
| **OptiLog** | 拜占庭共识中的角色分配优化 (Stavanger) |
| **DAG-based BFT** | 提升基于 DAG 的 BFT 状态机复制的吞吐量与可扩展性 (Supra/Purdue) |
| **Avicenna** | 通过反事实评估掩盖复制状态机中的慢节点 (Princeton/Databricks) |
| **Garen** | 基于原子状态调和的可靠集群管理 (FriendliAI) |
| **AEP** | 通过原子执行保护实现 DSM 层次化容错 (华科) |
| **Disaggregated Cache** | 面向复制存储系统的逻辑解耦缓存 (UIUC) |

**趋势:** BFT 共识正在以 DAG 架构和角色优化进行重新设计。FriendliAI 的 Garen 等来自工业界的集群管理可靠性论文体现了对正确性的关注。

---

### 十、可靠性与生产系统（约 4 篇）

| 论文 | 核心思路 |
|------|----------|
| **CSnake** | 通过因果拼接检测自维持级联故障 (Purdue) |
| **Proactive Change Risk Detection** | 字节跳动生产环境的变更风险主动检测经验 |
| **Rose** | 分布式系统中外部故障引发的失败重现 (INESC-ID/Purdue) |
| **Formal Methods at Huawei Cloud** | 华为云应用形式化方法的经验教训 |

**趋势:** 来自字节跳动和华为的生产经验论文标志着主动可靠性技术的成熟。级联故障检测和故障重现仍是难题。

---

### 十一、边缘/移动端 ML 与专用硬件（约 6 篇）

| 论文 | 核心思路 |
|------|----------|
| **FlexiQ** | 自适应混合精度量化 (首尔国立大学) |
| **viNPU** | 移动 NPU 上的 Vision Transformer 推理 (延世大学) |
| **Neuro-C** | 受硬件限制塑造的神经推理 (乌普萨拉大学) |
| **Efficient ML Model Updates** | 面向深度嵌入式微控制器的模型更新 (UC Berkeley) |
| **PointShuffler** | GPU 上加速点云神经网络 (中南大学) |
| **swKokkos** | 面向神威异构架构的 Kokkos 后端 (中科院) |

---

### 十二、其他亮点论文

| 论文 | 核心思路 |
|------|----------|
| **Ethane** | 基于紧凑 Trie 的区块链状态精简 (首尔国立大学) |
| **Carbon-Aware Learning** | 碳感知的可持续实时 ML 分析 (成均馆大学) |
| **E-Cube** | 面向无人机的事件增强高效视频流 (港大/清华) |
| **No More Translation at Runtime** | LLM 赋能的静态二进制翻译 (港科大) |
| **FlashPS** | 面向生成式图像编辑的掩码感知缓存与调度 (港科大/阿里巴巴) |
| **Matrix-PIC** | 高性能粒子模拟 (中山大学) |
| **Prediction-Informed Power Management** | 面向通用服务器的预测驱动功耗管理 (华盛顿大学) |
| **Million-Scale Video Retrieval** | 超维计算方法的百万级视频检索 (DGIST) |
| **GeDES** | GPU 驱动的离散事件网络仿真器 (电子科大) |

---

### 核心观察与趋势总结

**1. LLM 系统独占鳌头。** 约 30% 的录用论文直接针对 LLM 训练或服务，是最大的单一类别。社区已从 " 能不能服务 LLM" 转向 " 如何*高效、可靠、低成本地大规模服务*"。

**2. 生产规模成为新标杆。** 字节跳动 (MegaScale-MoE、网络故障处理、变更风险检测)、阿里巴巴 (Omar、LightDSA)、Microsoft (DROPS、REPS、Concord)、蚂蚁集团 (SKernel、Pyramid) 的论文展示了真实的大规模部署实践。

**3. 安全与 AI 融合。** TZ-LLM（TrustZone 保护 LLM）、TAO（神经网络验证）和 CofferOS（Rust 加固容器）标志着安全与 AI 系统的汇聚。

**4. 异构与解耦硬件。** SmartNIC 卸载 (BASK)、DPU 编译 (NutCracker)、GPU-FPGA DMA (RoPeerTo)、DSA 优化 (LightDSA) 和分层内存 (TierScape/MTTM) 反映了行业向专用解耦硬件迁移的方向。

**5. Serverless 走向成熟。** 7 篇 Serverless 论文——从成本分析到 GPU 中心数据传递再到生产画像——表明该模型正被应用于 AI 推理负载。

**6. 模糊测试成为系统方法论。** 三篇 Fuzzing 论文 (NecoFuzz 嵌套虚拟化、LifeFuzz 驱动、On-Hardware Fuzzing 嵌入式 OS) 确认模糊测试已成为系统正确性的标准手段。

**7. 亚洲（尤其中国）力量强势。** 多数论文涉及中国高校 (上交、清华、北大、华科等) 或中国企业 (字节、阿里、腾讯、蚂蚁、华为)，体现了该地区在系统研究上的巨大投入。

---

### 内核/虚拟化/安全方向重点推荐

- **CofferOS** — 基于 Rust 加固 OS 级虚拟化
- **NecoFuzz** — 嵌套虚拟化模糊测试（KVM 相关）
- **SKernel** — 分裂内核安全容器（内核架构）
- **PaCaR** — NUMA 上的 Page Cache 复制（内核内存管理）
- **Effective On-Hardware Fuzzing** — 嵌入式 OS 真实硬件模糊测试
- **BASK** — SmartNIC 卸载 KSM（内核同页合并卸载）
- **TZ-LLM** — 基于 TrustZone 的机密推理
- **VM Live Migration** — 异构处理器热迁移
- **Pomegranate** — 轻量级内核隔间化（arXiv 同期出现）

---

*数据来源: [EuroSys 2026 Accepted Papers](https://2026.eurosys.org/papers.html) | [ACM Proceedings](https://dl.acm.org/doi/proceedings/10.1145/3767295)*

<!-- obsidian-publish-source: Operating System 操作系统/Pappers 论文/Eurosys/EuroSys 2026 论文综述.md -->