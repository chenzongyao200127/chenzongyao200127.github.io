**Jeffrey Dean, Sanjay Ghemawat**
*Google, Inc.*

---

> 底层命题： 如何让完全不懂分布式系统、容错、网络调度的业务工程师，也能在成千上万台机器上编写并处理海量数据？
> 学习价值（抽象的威力）：将一切复杂的分布式计算抽象为 `Map` 和 `Reduce` 两个接口。尽管如今它的实现已被 Spark/Flink 取代，但它在“如何划定系统框架与用户逻辑的边界”、“如何通过设计彻底屏蔽底层硬件故障”上的第一性原理思考，至今无法被超越。

## 摘要

MapReduce 是一种用于处理和生成大规模数据集的编程模型（programming model）及其关联实现。用户指定一个 map 函数（映射函数）来处理一组 key/value pair（键/值对），以生成一组 intermediate key/value pairs（中间键/值对）；并指定一个 reduce 函数（归约函数），将所有与同一 intermediate key（中间键）相关联的 intermediate values（中间值）进行合并。如本文所示，现实世界中的许多任务都可以用这一模型来表达。以这种函数式风格编写的程序可以自动并行化，并在大规模 commodity machines（普通商用机器）集群上执行。运行时系统（run-time system）负责处理以下细节：对输入数据进行 partitioning（分区）、在一组机器上 scheduling（调度）程序的执行、处理 machine failures（机器故障），以及管理所需的 inter-machine communication（机器间通信）。这使得没有任何并行和分布式系统经验的程序员也能轻松利用大规模分布式系统的资源。我们的 MapReduce 实现运行在大规模普通商用机器集群上，具有高度的可扩展性（scalability）：一个典型的 MapReduce 计算可以在数千台机器上处理数 TB（terabytes）的数据。程序员发现这个系统非常易于使用：已有数百个 MapReduce 程序被实现，每天在 Google 的集群上执行超过一千个 MapReduce 作业。

---

## 1 引言

在过去五年中，本文作者及 Google 的许多同事实现了数百个专用计算程序，用于处理大量原始数据——例如爬取的文档、Web 请求日志等——以计算各种衍生数据，如 inverted indices（倒排索引）、Web 文档图结构的各种表示、每台主机爬取页面数量的汇总、某一天内最频繁查询的集合，等等。大多数这类计算在概念上是简单直接的。然而，输入数据通常非常庞大，为了在合理的时间内完成，计算必须分布在数百甚至数千台机器上执行。如何实现计算的并行化（parallelization）、数据的分发（data distribution）以及故障处理（failure handling）等问题，使得原本简单的计算被大量复杂的代码所掩盖。

为了应对这种复杂性，我们设计了一种新的抽象（abstraction），使我们能够表达希望执行的简单计算，同时将并行化、容错（fault-tolerance）、数据分发和负载均衡（load balancing）等繁杂细节隐藏在一个库中。我们的抽象受到了 Lisp 及许多其他函数式语言（functional languages）中 map 和 reduce 原语（primitives）的启发。我们意识到，我们的大多数计算都涉及对输入中的每条逻辑 " 记录 " 应用一个 map 操作，以计算一组中间键/值对，然后对所有共享同一键的值应用一个 reduce 操作，以适当地合并衍生数据。我们采用由用户指定 map 和 reduce 操作的函数式模型，使得大规模计算的并行化变得容易，并可以使用 re-execution（重新执行）作为容错的主要机制。

本工作的主要贡献在于：提供了一个简洁而强大的接口（interface），使大规模计算的自动并行化和分发成为可能；同时，该接口的实现在大规模普通商用 PC 集群上实现了高性能。第 2 节描述了基本的编程模型并给出了若干示例。第 3 节描述了针对我们基于集群的计算环境而定制的 MapReduce 接口实现。第 4 节描述了我们发现有用的编程模型的若干改进。第 5 节给出了我们的实现在各种任务上的性能测量结果。第 6 节探讨了 MapReduce 在 Google 内部的使用情况，包括我们将其作为生产索引系统（production indexing system）重写基础的经验。第 7 节讨论了相关工作和未来工作。

---

## 2 编程模型

计算接受一组输入键/值对，并产生一组输出键/值对。MapReduce 库的用户将计算表达为两个函数：**Map** 和 **Reduce**。

**Map** 由用户编写，接受一个输入对并产生一组中间键/值对。MapReduce 库将所有与同一中间键 *I* 相关联的中间值分组在一起，并将它们传递给 Reduce 函数。

**Reduce** 同样由用户编写，接受一个中间键 *I* 及该键对应的一组值。它将这些值合并，形成一个可能更小的值集合。通常每次 Reduce 调用只产生零个或一个输出值。中间值通过一个 iterator（迭代器）提供给用户的 reduce 函数，这使得我们能够处理因数据量过大而无法全部放入内存的值列表。

### 2.1 示例

考虑这样一个问题：在大量文档集合中统计每个单词的出现次数。用户将编写类似于以下伪代码的程序：

```shell
map(String key, String value):
  // key: document name
  // value: document contents
  for each word w in value:
    EmitIntermediate(w, "1");

reduce(String key, Iterator values):
  // key: a word
  // values: a list of counts
  int result = 0;
  for each v in values:
    result += ParseInt(v);
  Emit(AsString(result));
```

map 函数对每个单词及其关联的出现计数进行输出（在这个简单示例中计数仅为 "1"）。reduce 函数将针对某一特定单词输出的所有计数求和。

此外，用户编写代码来填充一个 mapreduce specification object（MapReduce 规约对象），其中包含输入和输出文件的名称，以及可选的调优参数。然后用户调用 MapReduce 函数，将该规约对象传入。用户的代码与 MapReduce 库（以 C++ 实现）链接在一起。附录 A 包含了此示例的完整程序文本。

### 2.2 类型

尽管前面的伪代码是用字符串输入和输出来编写的，但从概念上讲，用户提供的 map 和 reduce 函数具有如下关联类型：

> map (k1, v1) → list(k2, v2)
>
> reduce (k2, list(v2)) → list(v2)

即，输入的键和值来自与输出键和值不同的 domain（定义域）。此外，中间键和值与输出键和值来自相同的定义域。

我们的 C++ 实现中，向用户自定义函数传入和传出的均为字符串，由用户代码负责在字符串与适当类型之间进行转换。

### 2.3 更多示例

以下是一些简单但有趣的程序示例，它们可以方便地表达为 MapReduce 计算。

**Distributed Grep（分布式文本匹配）：** map 函数在某一行匹配给定 pattern（模式）时将其输出。reduce 函数是一个 identity function（恒等函数），仅将中间数据原样复制到输出。

**Count of URL Access Frequency（URL 访问频率统计）：** map 函数处理 Web 页面请求日志，输出 ⟨URL, 1⟩。reduce 函数将同一 URL 的所有值相加，输出 ⟨URL, total count⟩ 对。

**Reverse Web-Link Graph（反向 Web 链接图）：** map 函数对名为 *source* 的页面中发现的每一个指向目标 URL 的链接，输出 ⟨target, source⟩ 对。reduce 函数将与给定目标 URL 关联的所有源 URL 列表拼接起来，输出 ⟨target, list(source)⟩ 对。

**Term-Vector per Host（每主机词项向量）：** term vector（词项向量）将一个文档或一组文档中出现的最重要的词汇概括为 ⟨word, frequency⟩ 对的列表。map 函数为每个输入文档输出一个 ⟨hostname, term vector⟩ 对（其中 hostname 从文档的 URL 中提取）。reduce 函数接收给定主机的所有逐文档词项向量，将这些词项向量相加，丢弃低频词项，然后输出最终的 ⟨hostname, term vector⟩ 对。

**Inverted Index（倒排索引）：** map 函数解析每个文档，输出一系列 ⟨word, document ID⟩ 对。reduce 函数接收给定单词的所有对，对相应的 document ID 排序，输出 ⟨word, list(document ID)⟩ 对。所有输出对的集合构成一个简单的倒排索引。可以很容易地扩展此计算以记录单词位置信息。

**Distributed Sort（分布式排序）：** map 函数从每条记录中提取键，输出 ⟨key, record⟩ 对。reduce 函数将所有对原样输出。此计算依赖于第 4.1 节中描述的 partitioning facilities（分区机制）和第 4.2 节中描述的 ordering properties（排序特性）。

---

## 3 实现

MapReduce 接口有许多不同的实现方式，正确的选择取决于具体环境。例如，一种实现可能适用于小型共享内存（shared-memory）机器，另一种适用于大型 NUMA（非一致性内存访问）多处理器系统，还有一种则适用于规模更大的联网机器集群。

本节描述的实现面向 Google 内部广泛使用的计算环境：由交换式以太网（switched Ethernet）[4] 连接的大规模商用 PC 集群。在我们的环境中：

1. 机器通常配备双处理器 x86 处理器，运行 Linux 操作系统，每台机器拥有 2–4 GB 内存。
2. 使用商用网络硬件——单机层面通常为 100 兆比特/秒或 1 吉比特/秒，但整体对分带宽（bisection bandwidth）的平均值远低于此。
3. 集群由数百或数千台机器组成，因此机器故障十分常见。
4. 存储由直接连接到各台机器的廉价 IDE 磁盘提供。内部开发的分布式文件系统 [8] 用于管理存储在这些磁盘上的数据。该文件系统通过副本复制（replication）在不可靠的硬件之上提供可用性和可靠性。
5. 用户将作业（job）提交给调度系统。每个作业由一组任务（task）组成，由调度器（scheduler）映射到集群中的一组可用机器上。

### 3.1 执行概览

Map 调用通过将输入数据自动划分为 M 个分片（split）来分布到多台机器上。这些输入分片可以由不同的机器并行处理。Reduce 调用则通过使用分区函数（partitioning function）（例如 `hash(key) mod R`）将中间键空间（intermediate key space）划分为 R 个片段来进行分布。分区数量 R 及分区函数由用户指定。

图 1 展示了我们实现中 MapReduce 操作的整体流程。当用户程序调用 MapReduce 函数时，将发生以下一系列动作：

1. 用户程序中的 MapReduce 库首先将输入文件拆分为 M 个片段，每个片段通常为 16 MB 到 64 MB（用户可通过可选参数控制大小）。然后在集群的机器上启动该程序的多个副本。

2. 其中一个程序副本是特殊的——即 master（主节点）。其余的都是由 master 分配工作的 worker（工作节点）。共有 M 个 map 任务和 R 个 reduce 任务需要分配。master 选取空闲的 worker，为每个 worker 分配一个 map 任务或一个 reduce 任务。

3. 被分配了 map 任务的 worker 读取对应输入分片的内容，从输入数据中解析出键/值对（key/value pair），并将每个键/值对传递给用户定义的 Map 函数。Map 函数产生的中间键/值对被缓存在内存中。

4. 这些缓存的键/值对会被周期性地写入本地磁盘，并由分区函数划分为 R 个区域（region）。这些缓存数据在本地磁盘上的位置被回传给 master，由 master 负责将这些位置信息转发给 reduce worker。

5. 当 reduce worker 收到 master 发来的位置通知后，它通过远程过程调用（remote procedure call, RPC）从 map worker 的本地磁盘上读取缓存数据。当 reduce worker 读完所有中间数据后，它会按中间键进行排序，使得同一键的所有出现被聚集在一起。排序是必要的，因为通常有许多不同的键映射到同一个 reduce 任务。如果中间数据量过大而无法全部放入内存，则使用外部排序（external sort）。

6. reduce worker 遍历排序后的中间数据，对于遇到的每个唯一中间键，将该键及其对应的中间值集合传递给用户的 Reduce 函数。Reduce 函数的输出被追加到该 reduce 分区对应的最终输出文件中。

7. 当所有 map 任务和 reduce 任务完成后，master 唤醒用户程序。此时，用户程序中的 MapReduce 调用返回到用户代码。

成功完成后，MapReduce 执行的输出存放在 R 个输出文件中（每个 reduce 任务对应一个文件，文件名由用户指定）。通常，用户不需要将这 R 个输出文件合并为一个文件——他们往往将这些文件作为另一次 MapReduce 调用的输入，或者在另一个能够处理多文件分区输入的分布式应用程序中使用它们。

### 3.2 Master 数据结构

master 维护若干数据结构。对于每个 map 任务和 reduce 任务，它存储其状态（空闲 idle、进行中 in-progress 或已完成 completed）以及 worker 机器的标识（针对非空闲任务）。

master 是中间文件区域位置信息从 map 任务传播到 reduce 任务的管道（conduit）。因此，对于每个已完成的 map 任务，master 存储该 map 任务产生的 R 个中间文件区域的位置和大小。这些位置和大小信息在 map 任务完成时被接收并更新，并被增量地推送给正在执行 reduce 任务的 worker。

### 3.3 容错

由于 MapReduce 库被设计用于在数百或数千台机器上处理海量数据，因此该库必须能够优雅地容忍机器故障。

#### Worker 故障

master 周期性地对每个 worker 进行 ping 探测。如果在一定时间内未收到某个 worker 的响应，master 将该 worker 标记为失败（failed）。该 worker 已完成的所有 map 任务将被重置为初始的空闲状态，从而可以在其他 worker 上重新调度。类似地，在失败 worker 上进行中的任何 map 任务或 reduce 任务也会被重置为空闲状态，并可被重新调度。

已完成的 map 任务需要重新执行，因为其输出存储在失败机器的本地磁盘上，因而不可访问。已完成的 reduce 任务则无需重新执行，因为其输出存储在全局文件系统（global file system）中。

当一个 map 任务先由 worker A 执行，后来又由 worker B 执行（因为 A 失败了）时，所有正在执行 reduce 任务的 worker 都会收到重新执行的通知。任何尚未从 worker A 读取数据的 reduce 任务将改为从 worker B 读取数据。

MapReduce 能够抵御大规模 worker 故障。例如，在一次 MapReduce 操作过程中，正在运行的集群上的网络维护导致每次有 80 台机器同时变得不可达，持续数分钟。MapReduce 的 master 只是简单地重新执行那些不可达 worker 机器上已完成的工作，并继续向前推进，最终完成了 MapReduce 操作。

#### Master 故障

让 master 周期性地对上述 master 数据结构写入检查点（checkpoint）是很容易的。如果 master 任务终止，可以从最近的检查点状态启动一个新的副本。然而，鉴于 master 只有一个，其发生故障的概率很低；因此，我们当前的实现在 master 失败时直接终止（abort）MapReduce 计算。客户端可以检测此状况，并在需要时重试 MapReduce 操作。

#### 故障存在时的语义

当用户提供的 map 和 reduce 算子（operator）是其输入值的确定性函数（deterministic function）时，我们的分布式实现所产生的输出与整个程序的一次无故障顺序执行（non-faulting sequential execution）所产生的输出相同。

我们依赖 map 和 reduce 任务输出的原子提交（atomic commit）来实现这一性质。每个进行中的任务将其输出写入私有临时文件。一个 reduce 任务产生一个这样的文件，一个 map 任务产生 R 个这样的文件（每个 reduce 任务对应一个）。当一个 map 任务完成时，worker 向 master 发送一条消息，消息中包含 R 个临时文件的名称。如果 master 收到一个已完成 map 任务的重复完成消息，它会忽略该消息。否则，它将 R 个文件的名称记录在 master 数据结构中。

当一个 reduce 任务完成时，reduce worker 将其临时输出文件原子性地重命名（atomically rename）为最终输出文件。如果同一个 reduce 任务在多台机器上执行，则会对同一最终输出文件执行多次重命名调用。我们依赖底层文件系统提供的原子重命名操作来保证最终的文件系统状态只包含 reduce 任务的一次执行所产生的数据。

我们的绝大多数 map 和 reduce 算子都是确定性的，在此情况下我们的语义等价于顺序执行，这使得程序员能够非常容易地推理其程序的行为。当 map 和/或 reduce 算子是非确定性的（non-deterministic）时，我们提供较弱但仍然合理的语义。在非确定性算子存在的情况下，某个特定 reduce 任务 R1 的输出等价于该非确定性程序的某次顺序执行所产生的 R1 的输出。然而，另一个 reduce 任务 R2 的输出可能对应于该非确定性程序的另一次不同顺序执行所产生的 R2 的输出。

考虑 map 任务 M 和 reduce 任务 R1、R2。设 e(Ri) 为 Ri 已提交的那次执行（恰好存在一次这样的执行）。较弱语义的产生是因为 e(R1) 可能读取了 M 的某次执行所产生的输出，而 e(R2) 可能读取了 M 的另一次不同执行所产生的输出。

### 3.4 数据本地性

在我们的计算环境中，网络带宽是相对稀缺的资源。我们利用以下事实来节省网络带宽：输入数据（由 GFS [8] 管理）存储在组成集群的机器的本地磁盘上。GFS 将每个文件分割为 64 MB 的块（block），并在不同机器上存储每个块的若干副本（通常为 3 个副本）。MapReduce 的 master 会考虑输入文件的位置信息，尝试在包含对应输入数据副本的机器上调度 map 任务。如果无法做到这一点，它会尝试在靠近该任务输入数据副本的机器上调度 map 任务（例如，在与存储数据的机器位于同一网络交换机的 worker 机器上）。在集群中相当大比例的 worker 上运行大规模 MapReduce 操作时，大部分输入数据在本地读取，不消耗网络带宽。

### 3.5 任务粒度

如前所述，我们将 map 阶段细分为 M 个片段，reduce 阶段细分为 R 个片段。理想情况下，M 和 R 应远大于 worker 机器的数量。让每个 worker 执行许多不同的任务可以改善动态负载均衡（dynamic load balancing），并在 worker 发生故障时加快恢复速度：故障 worker 已完成的众多 map 任务可以分散到所有其他 worker 机器上重新执行。

在我们的实现中，M 和 R 的大小存在实际的上限，因为如前所述，master 必须做出 O(M + R) 个调度决策，并在内存中保持 O(M * R) 的状态。（不过内存使用的常数因子很小：O(M * R) 部分的状态大约为每个 map 任务/reduce 任务对一个字节的数据。）

此外，R 通常受到用户的约束，因为每个 reduce 任务的输出最终会生成一个单独的输出文件。在实践中，我们倾向于选择 M 使得每个单独的任务大约对应 16 MB 到 64 MB 的输入数据（以使上述数据本地性优化最为有效），并使 R 为我们预期使用的 worker 机器数量的较小倍数。我们通常使用 2,000 台 worker 机器执行 M = 200,000、R = 5,000 的 MapReduce 计算。

### 3.6 备用任务

导致 MapReduce 操作总耗时延长的一个常见原因是 " 落伍者 "（straggler）：某台机器在计算接近尾声时，花费异常长的时间来完成最后几个 map 或 reduce 任务之一。落伍者的产生可能有多种原因。例如，一台磁盘存在缺陷的机器可能频繁遇到可纠正错误，导致其读取性能从 30 MB/s 降至 1 MB/s。集群调度系统可能在该机器上调度了其他任务，导致其因 CPU、内存、本地磁盘或网络带宽的竞争而更慢地执行 MapReduce 代码。我们近期遇到的一个问题是机器初始化代码中的一个 bug 导致处理器缓存（processor cache）被禁用：受影响机器上的计算速度下降了超过百倍。

我们有一种通用机制来缓解落伍者问题。当一个 MapReduce 操作接近完成时，master 为剩余的进行中任务调度备用执行（backup execution）。无论是主执行还是备用执行完成，该任务即被标记为已完成。我们已对此机制进行了调优，使其通常仅增加不超过百分之几的计算资源消耗。我们发现这显著缩短了大规模 MapReduce 操作的完成时间。例如，第 5.3 节描述的排序程序在禁用备用任务机制后，完成时间延长了 44%。

---

## 4 改进（Refinements）

尽管仅通过编写 Map 和 Reduce 函数所提供的基本功能已能满足大多数需求，但我们发现了一些有用的扩展。本节将对这些扩展进行描述。

### 4.1 分区函数（Partitioning Function）

MapReduce 的用户需要指定所期望的 Reduce 任务/输出文件的数量（R）。数据通过一个作用于中间键（intermediate key）上的分区函数被分配到这些任务中。系统提供了一个默认的分区函数，该函数使用哈希（hashing）方法（例如 `hash(key) mod R`）。这种方式通常能产生相当均衡的分区。然而，在某些情况下，使用键的其他函数来对数据进行分区是有用的。例如，有时输出键是 URL，而我们希望同一主机（host）的所有条目最终出现在同一个输出文件中。为了支持此类场景，MapReduce 库的用户可以提供一个自定义的分区函数。例如，使用 `hash(Hostname(urlkey)) mod R` 作为分区函数，可以使来自同一主机的所有 URL 最终进入同一个输出文件。

### 4.2 顺序保证（Ordering Guarantees）

我们保证在给定的分区内，中间键/值对（intermediate key/value pairs）按键的递增顺序进行处理。这一排序保证使得每个分区能够方便地生成有序的输出文件，这在输出文件格式需要支持按键进行高效随机访问查找（random access lookups）时非常有用，或者当输出数据的使用者希望数据是有序的时候也很方便。

### 4.3 合并函数（Combiner Function）

在某些情况下，每个 Map 任务产生的中间键存在大量重复，且用户指定的 Reduce 函数满足交换律（commutative）和结合律（associative）。一个典型的例子是 Section 2.1 中的单词计数（word counting）示例。由于词频倾向于遵循齐普夫分布（Zipf distribution），每个 Map 任务将产生成百上千条形如 <the, 1> 的记录。所有这些计数将通过网络发送到单个 Reduce 任务，然后由 Reduce 函数将它们相加以产生一个最终数值。我们允许用户指定一个可选的合并函数（Combiner function），在数据通过网络发送之前对其进行部分合并（partial merging）。

合并函数在每台执行 Map 任务的机器上运行。通常，合并函数和 Reduce 函数使用相同的代码实现。Reduce 函数与合并函数之间的唯一区别在于 MapReduce 库对函数输出的处理方式不同。Reduce 函数的输出被写入最终输出文件。合并函数的输出则被写入一个中间文件（intermediate file），该文件随后将被发送到 Reduce 任务。

部分合并能够显著加速某些类别的 MapReduce 操作。附录 A 中包含了一个使用合并函数的示例。

### 4.4 输入与输出类型（Input and Output Types）

MapReduce 库提供了对多种不同格式输入数据读取的支持。例如，"text" 模式的输入将每一行视为一个键/值对：键是该行在文件中的偏移量（offset），值是该行的内容。另一种常见的受支持格式存储的是按键排序的键/值对序列。每种输入类型（input type）的实现都知道如何将自身分割为有意义的范围（range），以作为独立的 Map 任务进行处理（例如，text 模式的范围分割确保分割仅发生在行边界处）。用户可以通过提供一个简单的读取器接口（reader interface）的实现来添加对新输入类型的支持，不过大多数用户只使用少数几种预定义的输入类型。

读取器不一定需要提供从文件中读取的数据。例如，可以很容易地定义一个从数据库中读取记录的读取器，或者从映射在内存中的数据结构中读取记录的读取器。

类似地，我们也支持一组输出类型（output types），用于以不同格式生成数据，并且用户代码可以方便地添加对新输出类型的支持。

### 4.5 副作用（Side-effects）

在某些情况下，MapReduce 的用户发现从其 Map 和/或 Reduce 操作中生成辅助文件（auxiliary files）作为额外输出是很方便的。我们依赖应用程序编写者来确保此类副作用具有原子性（atomic）和幂等性（idempotent）。通常，应用程序先写入一个临时文件，待文件完全生成后再对其进行原子性重命名。

我们不提供对单个任务产生的多个输出文件进行原子两阶段提交（atomic two-phase commits）的支持。因此，产生多个输出文件且存在跨文件一致性（cross-file consistency）要求的任务应当是确定性的（deterministic）。这一限制在实践中从未成为问题。

### 4.6 跳过损坏记录（Skipping Bad Records）

有时用户代码中存在缺陷（bug），导致 Map 或 Reduce 函数在处理某些记录时确定性地崩溃（crash deterministically）。此类缺陷会阻止 MapReduce 操作的完成。通常的做法是修复该缺陷，但有时这并不可行——例如，该缺陷可能存在于源代码不可获取的第三方库中。此外，有时忽略少量记录是可以接受的，例如在对大规模数据集进行统计分析时。我们提供了一种可选的执行模式，在该模式下 MapReduce 库能够检测出哪些记录会导致确定性崩溃，并跳过这些记录以确保计算能够继续推进。

每个工作进程（worker process）安装一个信号处理器（signal handler），用于捕获段错误（segmentation violations）和总线错误（bus errors）。在调用用户的 Map 或 Reduce 操作之前，MapReduce 库将参数的序列号（sequence number）存储在一个全局变量中。如果用户代码产生了一个信号，信号处理器将发送一个包含该序列号的 " 临终遗言 "（last gasp）UDP 数据包给 MapReduce 主节点（master）。当主节点发现某条特定记录导致了不止一次失败时，它会在下一次重新执行相应的 Map 或 Reduce 任务时指示应跳过该记录。

### 4.7 本地执行（Local Execution）

在 Map 或 Reduce 函数中调试问题可能相当棘手，因为实际计算发生在一个分布式系统中，通常运行在数千台机器上，且工作分配决策由主节点动态做出。为了便于调试、性能分析（profiling）和小规模测试，我们开发了 MapReduce 库的一个替代实现，该实现在本地机器上顺序地（sequentially）执行 MapReduce 操作的所有工作。用户可以通过控制参数将计算限制在特定的 Map 任务上。用户使用特殊的标志调用其程序，随后可以方便地使用任何他们认为有用的调试或测试工具（例如 gdb）。

### 4.8 状态信息（Status Information）

主节点（Master）内部运行一个 HTTP 服务器，并导出一组供用户查阅的状态页面（Status Pages）。这些状态页面显示计算的进度信息，例如已完成的任务数量、正在进行中的任务数量、输入数据的字节数、中间数据（Intermediate Data）的字节数、输出数据的字节数、处理速率等。页面中还包含指向每个任务所生成的标准错误输出（Standard Error）和标准输出（Standard Output）文件的链接。用户可以利用这些数据来预测计算还需要多长时间，以及是否需要为计算投入更多资源。这些页面还可以用于排查计算速度远低于预期的情况。

此外，顶层状态页面还会显示哪些工作节点（Worker）已经失败，以及它们失败时正在处理的 Map 任务和 Reduce 任务。这些信息在尝试诊断用户代码中的缺陷时非常有用。

### 4.9 计数器（Counters）

MapReduce 库提供了一个计数器机制（Counter Facility），用于统计各种事件的发生次数。例如，用户代码可能希望统计已处理的单词总数，或者已索引的德语文档数量等。

要使用该机制，用户代码需要创建一个命名的计数器对象（Counter Object），然后在 Map 和/或 Reduce 函数中适当地递增该计数器。例如：

```shell
Counter* uppercase;
uppercase = GetCounter("uppercase");

map(String name, String contents):
  for each word w in contents:
    if (IsCapitalized(w)):
      uppercase->Increment();
    EmitIntermediate(w, "1");
```

各个工作节点机器上的计数器值会被周期性地传播到主节点（附带在心跳响应（Ping Response）中）。主节点汇总来自所有成功完成的 Map 任务和 Reduce 任务的计数器值，并在 MapReduce 操作完成时将其返回给用户代码。当前的计数器值也会显示在主节点的状态页面上，以便用户可以实时观察计算的进展。在汇总计数器值时，主节点会消除同一 Map 或 Reduce 任务重复执行所带来的影响，以避免重复计数（Double Counting）。（重复执行可能源于备用任务（Backup Tasks）的使用，以及因故障而重新执行任务。）

MapReduce 库会自动维护某些计数器值，例如已处理的输入键/值对（Input Key/Value Pairs）的数量和已产生的输出键/值对（Output Key/Value Pairs）的数量。

用户发现计数器机制对于验证 MapReduce 操作行为的正确性（Sanity Checking）非常有用。例如，在某些 MapReduce 操作中，用户代码可能希望确保产生的输出对的数量恰好等于已处理的输入对的数量，或者确保已处理的德语文档占全部已处理文档总数的比例在某个可容忍的范围之内。

---

## 5 性能（Performance）

在本节中，我们在一个大规模机器集群上，通过两个计算任务来衡量 MapReduce 的性能。其中一个计算任务在大约一太字节（terabyte）的数据中搜索特定模式（pattern）；另一个计算任务对大约一太字节的数据进行排序。

这两个程序代表了 MapReduce 用户所编写的真实程序中的一大类——一类程序将数据从一种表示形式转换（shuffle）为另一种表示形式，另一类程序则从大规模数据集中提取少量有价值的数据。

### 5.1 集群配置（Cluster Configuration）

所有程序均在一个由大约 1800 台机器组成的集群上执行。每台机器配备两个启用了超线程（Hyper-Threading）的 2GHz Intel Xeon 处理器、4GB 内存、两块 160GB IDE 硬盘，以及一条千兆以太网（gigabit Ethernet）链路。这些机器通过一个两层树形交换网络（two-level tree-shaped switched network）连接，根节点处的聚合带宽（aggregate bandwidth）约为 100–200 Gbps。所有机器位于同一托管设施（hosting facility）中，因此任意两台机器之间的往返时间（round-trip time）均小于一毫秒。

在 4GB 内存中，大约有 1–1.5GB 被集群上运行的其他任务所占用。程序在一个周末下午执行，此时 CPU、磁盘和网络大多处于空闲状态。

### 5.2 Grep

grep 程序扫描 10^10 条 100 字节的记录，搜索一个相对罕见的三字符模式（该模式出现在 92,337 条记录中）。输入数据被划分为大约 64MB 的分片（M = 15000），全部输出写入一个文件（R = 1）。

Figure 2 展示了计算过程随时间的推进情况。Y 轴表示输入数据的扫描速率。随着越来越多的机器被分配到该 MapReduce 计算任务，速率逐渐上升，当 1764 个工作节点（worker）被分配后，峰值超过 30 GB/s。随着 map 任务陆续完成，速率开始下降，并在计算开始约 80 秒后降至零。整个计算从开始到结束大约耗时 150 秒，其中包括大约一分钟的启动开销（startup overhead）。该开销源于将程序分发到所有工作节点机器的过程，以及与 GFS 交互以打开 1000 个输入文件集合并获取局部性优化（locality optimization）所需信息的延迟。

### 5.3 排序（Sort）

排序程序对 10^10 条 100 字节的记录（约 1 太字节数据）进行排序。该程序以 TeraSort 基准测试（benchmark）[10] 为原型。

排序程序的用户代码不到 50 行。一个三行的 Map 函数从文本行中提取 10 字节的排序键（sorting key），并将该键与原始文本行作为中间键/值对（intermediate key/value pair）输出。我们使用内置的恒等函数（Identity function）作为 Reduce 算子（operator），该函数将中间键/值对原样传递为输出键/值对。最终的排序输出被写入一组两路复制（2-way replicated）的 GFS 文件（即程序的输出共写入 2 太字节的数据）。

与之前一样，输入数据被划分为 64MB 的分片（M = 15000）。我们将排序输出分为 4000 个文件（R = 4000）。分区函数（partitioning function）利用键的起始字节将其分配到 R 个分片之一。

在本基准测试中，我们的分区函数内置了键分布（distribution of keys）的先验知识。在通用的排序程序中，我们会增加一个预处理（pre-pass）的 MapReduce 操作来采样键的分布，并利用采样键的分布计算最终排序阶段的分割点（splitpoints）。

Figure 3 (a) 展示了排序程序正常执行的过程。左上方的图展示了输入数据的读取速率。速率峰值约为 13 GB/s，随后迅速下降，因为所有 map 任务在 200 秒之前已经完成。需要注意的是，输入速率低于 grep 程序，这是因为排序的 map 任务将大约一半的时间和 I/O 带宽用于将中间输出写入本地磁盘。相比之下，grep 的中间输出大小可以忽略不计。

左侧中间的图展示了数据从 map 任务通过网络传输到 reduce 任务的速率。这一混洗（shuffling）过程在第一个 map 任务完成后即刻开始。图中的第一个峰包（hump）对应第一批约 1700 个 reduce 任务（整个 MapReduce 计算被分配了约 1700 台机器，每台机器同一时刻最多执行一个 reduce 任务）。在计算进行到大约 300 秒时，第一批 reduce 任务中的部分任务已经完成，系统开始为剩余的 reduce 任务混洗数据。所有混洗工作在计算进行到约 600 秒时完成。

左下方的图展示了 reduce 任务将排序后的数据写入最终输出文件的速率。在第一个混洗阶段结束与写入阶段开始之间存在一段延迟，这是因为机器正忙于对中间数据进行排序。写入速率在一段时间内维持在约 2–4 GB/s。所有写入工作在计算进行到约 850 秒时完成。加上启动开销，整个计算耗时 891 秒。这与 TeraSort 基准测试当前报告的最佳结果 1057 秒 [18] 相近。

有几点值得注意：输入速率高于混洗速率和输出速率，这得益于我们的局部性优化——大部分数据从本地磁盘读取，从而绕过了带宽受限的网络。混洗速率高于输出速率，是因为输出阶段需要写入排序数据的两份副本（出于可靠性和可用性的考虑，我们对输出进行双副本复制）。我们写入两份副本是因为这是底层文件系统提供的可靠性和可用性保障机制。如果底层文件系统采用纠删码（erasure coding）[14] 而非复制（replication），则写入数据所需的网络带宽将会降低。

### 5.4 备用任务的效果（Effect of Backup Tasks）

在 Figure 3 (b) 中，我们展示了禁用备用任务（backup tasks）后排序程序的执行情况。执行流程与 Figure 3 (a) 所示类似，但在尾部出现了一段很长的时期，期间几乎没有写入活动。在 960 秒后，除了 5 个 reduce 任务之外，其余全部完成。然而，这最后几个落伍者（stragglers）直到 300 秒之后才完成。整个计算耗时 1283 秒，总用时增加了 44%。

### 5.5 机器故障（Machine Failures）

在 Figure 3 (c) 中，我们展示了排序程序的一次执行，在计算开始几分钟后，我们故意终止了 1746 个工作进程中的 200 个。底层集群调度器（cluster scheduler）立即在这些机器上重新启动了新的工作进程（由于仅终止了进程，机器本身仍在正常运行）。

工作进程的终止表现为负的输入速率，这是因为一些先前已完成的 map 工作丢失了（由于相应的 map 工作节点被终止），需要重新执行。这些 map 工作的重新执行（re-execution）相当迅速。包含启动开销在内，整个计算在 933 秒内完成（仅比正常执行时间增加了 5%）。

---

## 6 经验（Experience）

我们于 2003 年 2 月编写了 MapReduce 库的第一个版本，并在 2003 年 8 月对其进行了重大改进，包括本地性优化（locality optimization）、工作机器间任务执行的动态负载均衡（dynamic load balancing）等。自那时起，我们惊喜地发现 MapReduce 库在我们所处理的各类问题上具有广泛的适用性。它已在 Google 内部的多个领域得到应用，包括：

- 大规模机器学习（machine learning）问题，
- Google News 和 Froogle 产品的聚类（clustering）问题，
- 提取用于生成热门查询报告的数据（例如 Google Zeitgeist），
- 提取网页属性以用于新的实验和产品（例如从大规模网页语料库中提取地理位置信息以支持本地化搜索），以及
- 大规模图计算（graph computations）。

图 4 展示了在我们主要的源代码管理系统中，独立的 MapReduce 程序数量随时间推移的显著增长——从 2003 年初的 0 个增长到 2004 年 9 月底的近 900 个独立实例。MapReduce 之所以如此成功，是因为它使得编写一个简单程序并在半小时内高效地运行于上千台机器上成为可能，从而极大地加速了开发和原型设计的周期。此外，它使得没有分布式和/或并行系统经验的程序员也能轻松利用大量资源。

在每个作业（job）结束时，MapReduce 库会记录该作业所使用的计算资源的统计信息。在表 1 中，我们展示了 2004 年 8 月在 Google 运行的部分 MapReduce 作业的一些统计数据。

**表 1：2004 年 8 月运行的 MapReduce 作业**

| 统计项 | 数值 |
|---|---|
| 作业数量（Number of jobs） | 29,423 |
| 平均作业完成时间（Average job completion time） | 634 秒 |
| 使用的机器天数（Machine days used） | 79,186 天 |
| 读取的输入数据（Input data read） | 3,288 TB |
| 产生的中间数据（Intermediate data produced） | 758 TB |
| 写入的输出数据（Output data written） | 193 TB |
| 每个作业的平均工作机器数（Average worker machines per job） | 157 |
| 每个作业的平均工作机器故障数（Average worker deaths per job） | 1.2 |
| 每个作业的平均 Map 任务数（Average map tasks per job） | 3,351 |
| 每个作业的平均 Reduce 任务数（Average reduce tasks per job） | 55 |
| 独立的 Map 实现数（Unique map implementations） | 395 |
| 独立的 Reduce 实现数（Unique reduce implementations） | 269 |
| 独立的 Map/Reduce 组合数（Unique map/reduce combinations） | 426 |

### 6.1 大规模索引构建（Large-Scale Indexing）

迄今为止，我们对 MapReduce 最重要的应用之一是对生产环境索引系统（production indexing system）的完全重写，该系统负责生成 Google 网页搜索服务所使用的数据结构。索引系统以我们的爬取系统（crawling system）所获取的大量文档集合作为输入，这些文档以一组 GFS 文件的形式存储。这些文档的原始内容超过 20 太字节（terabytes）的数据。索引过程以五到十个 MapReduce 操作的序列运行。使用 MapReduce（而非索引系统先前版本中的临时性分布式处理流程（ad-hoc distributed passes））带来了以下几个好处：

- 索引代码变得更简单、更精简且更易于理解，因为处理容错（fault tolerance）、分布式和并行化的代码被隐藏在 MapReduce 库内部。例如，计算的某一阶段的代码规模从约 3800 行 C++ 代码缩减到使用 MapReduce 表达时的约 700 行。

- MapReduce 库的性能足够优秀，使得我们可以将概念上不相关的计算保持分离，而无需将它们混在一起以避免对数据的额外遍历。这使得修改索引过程变得容易。例如，在旧索引系统中需要花费数月才能完成的一项修改，在新系统中仅需几天即可实现。

- 索引过程的运维（operate）变得更加容易，因为大多数由机器故障、慢速机器和网络波动引起的问题都由 MapReduce 库自动处理，无需操作人员干预。此外，通过向索引集群中添加新机器，可以轻松提升索引过程的性能。

---

## 7 相关工作（Related Work）

许多系统提供了受限的编程模型（restricted programming models），并利用这些限制来自动并行化计算。例如，利用并行前缀计算（parallel prefix computations）[6, 9, 13]，可以在 *N* 个处理器上以 log *N* 的时间计算一个 *N* 元素数组所有前缀上的结合函数（associative function）。MapReduce 可以被视为基于我们在大规模实际计算中的经验，对这些模型的一种简化和提炼。更重要的是，我们提供了一个可扩展到数千个处理器的容错实现。相比之下，大多数并行处理系统仅在较小规模上实现，并将处理机器故障的细节留给了程序员。

批量同步编程（Bulk Synchronous Programming）[17] 和一些 MPI 原语（MPI primitives）[11] 提供了更高层次的抽象，使程序员更容易编写并行程序。这些系统与 MapReduce 之间的一个关键区别在于，MapReduce 利用受限的编程模型来自动并行化用户程序，并提供透明的容错机制。

我们的本地性优化（locality optimization）借鉴了活跃磁盘（active disks）[12, 15] 等技术的思想，这些技术将计算推送到靠近本地磁盘的处理单元上，以减少通过 I/O 子系统或网络传输的数据量。我们运行在直接连接少量磁盘的商用处理器上，而非直接运行在磁盘控制器处理器上，但总体思路是类似的。

我们的备份任务机制（backup task mechanism）类似于 Charlotte 系统 [3] 中所采用的急切调度机制（eager scheduling mechanism）。简单急切调度的一个缺点是，如果某个给定任务反复导致失败，则整个计算将无法完成。我们通过跳过坏记录（skipping bad records）的机制修复了此类问题的部分情形。

MapReduce 的实现依赖于一个内部的集群管理系统（cluster management system），该系统负责在大量共享机器上分发和运行用户任务。虽然不是本文的重点，但该集群管理系统在设计理念上与 Condor [16] 等其他系统相似。

作为 MapReduce 库一部分的排序功能在操作方式上类似于 NOW-Sort [1]。源机器（Map 工作机器）将待排序的数据进行分区，并发送到 *R* 个 Reduce 工作机器之一。每个 Reduce 工作机器在本地对其数据进行排序（如果可能则在内存中进行）。当然，NOW-Sort 没有用户可定义的 Map 和 Reduce 函数，而正是这些函数使得我们的库具有广泛的适用性。

River [2] 提供了一种编程模型，其中进程通过分布式队列（distributed queues）相互发送数据进行通信。与 MapReduce 类似，River 系统也试图在异构硬件或系统扰动引入的不均匀性（non-uniformities）存在的情况下，提供良好的平均性能。River 通过精心调度磁盘和网络传输来实现均衡的完成时间。MapReduce 采用了不同的方法。通过限制编程模型，MapReduce 框架能够将问题划分为大量细粒度（fine-grained）任务。这些任务被动态调度到可用的工作机器上，使得较快的工作机器处理更多的任务。受限的编程模型还允许我们在作业接近结束时调度任务的冗余执行（redundant executions），这在存在不均匀性（如慢速或卡住的工作机器）时大大减少了完成时间。

BAD-FS [5] 的编程模型与 MapReduce 截然不同，并且与 MapReduce 不同，它面向跨广域网络（wide-area network）的作业执行。然而，两者存在两个基本的相似之处：(1) 两个系统都使用冗余执行来从因故障导致的数据丢失中恢复；(2) 两者都使用本地性感知调度（locality-aware scheduling）来减少通过拥塞网络链路发送的数据量。

TACC [7] 是一个旨在简化高可用网络服务（highly-available networked services）构建的系统。与 MapReduce 类似，它依赖重新执行（re-execution）作为实现容错的机制。

---

## 8 结论（Conclusions）

MapReduce 编程模型已在 Google 被成功地应用于许多不同的用途。我们将这一成功归因于几个原因。首先，该模型易于使用，即使对于没有并行和分布式系统经验的程序员也是如此，因为它隐藏了并行化、容错、本地性优化和负载均衡的细节。其次，大量多样的问题可以轻松地表示为 MapReduce 计算。例如，MapReduce 被用于为 Google 的生产环境网页搜索服务生成数据、排序、数据挖掘（data mining）、机器学习以及许多其他系统。第三，我们开发了一个可扩展到由数千台机器组成的大型集群的 MapReduce 实现。该实现高效地利用了这些机器资源，因此适用于 Google 所面临的许多大规模计算问题。

我们从这项工作中获得了一些认识。首先，限制编程模型使得并行化和分布式计算以及使此类计算具有容错性变得容易。其次，网络带宽（network bandwidth）是一种稀缺资源。因此，我们系统中的许多优化都旨在减少通过网络发送的数据量：本地性优化使我们能够从本地磁盘读取数据，而将中间数据的单个副本写入本地磁盘则节省了网络带宽。第三，冗余执行可用于减少慢速机器的影响，并处理机器故障和数据丢失。

---

## 致谢（Acknowledgements）

Josh Levenberg 在修订和扩展用户级 MapReduce API 方面发挥了重要作用，基于他使用 MapReduce 的经验以及其他人对增强功能的建议，增加了许多新特性。MapReduce 从 Google 文件系统（Google File System）[8] 中读取输入并向其写入输出。我们要感谢 Mohit Aron、Howard Gobioff、Markus Gutschke、David Kramer、Shun-Tak Leung 和 Josh Redstone 在开发 GFS 方面所做的工作。我们还要感谢 Percy Liang 和 Olcan Sercinoglu 在开发 MapReduce 所使用的集群管理系统方面所做的工作。Mike Burrows、Wilson Hsieh、Josh Levenberg、Sharon Perl、Rob Pike 和 Debby Wallach 对本文的早期草稿提供了有益的评论。匿名的 OSDI 审稿人以及我们的指导人（shepherd）Eric Brewer 提供了许多有用的建议，指出了论文可以改进的方面。

---

## 参考文献（References）

[1] Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, David E. Culler, Joseph M. Hellerstein, and David A. Patterson. High-performance sorting on networks of workstations. In *Proceedings of the 1997 ACM SIGMOD International Conference on Management of Data*, Tucson, Arizona, May 1997.

[2] Remzi H. Arpaci-Dusseau, Eric Anderson, Noah Treuhaft, David E. Culler, Joseph M. Hellerstein, David Patterson, and Kathy Yelick. Cluster I/O with River: Making the fast case common. In *Proceedings of the Sixth Workshop on Input/Output in Parallel and Distributed Systems (IOPADS '99)*, pages 10–22, Atlanta, Georgia, May 1999.

[3] Arash Baratloo, Mehmet Karaul, Zvi Kedem, and Peter Wyckoff. Charlotte: Metacomputing on the web. In *Proceedings of the 9th International Conference on Parallel and Distributed Computing Systems*, 1996.

[4] Luiz A. Barroso, Jeffrey Dean, and Urs Hölzle. Web search for a planet: The Google cluster architecture. *IEEE Micro*, 23(2):22–28, April 2003.

[5] John Bent, Douglas Thain, Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, and Miron Livny. Explicit control in a batch-aware distributed file system. In *Proceedings of the 1st USENIX Symposium on Networked Systems Design and Implementation (NSDI)*, March 2004.

[6] Guy E. Blelloch. Scans as primitive parallel operations. *IEEE Transactions on Computers*, C-38(11):1526–1538, November 1989.

[7] Armando Fox, Steven D. Gribble, Yatin Chawathe, Eric A. Brewer, and Paul Gauthier. Cluster-based scalable network services. In *Proceedings of the 16th ACM Symposium on Operating System Principles*, pages 78–91, Saint-Malo, France, 1997.

[8] Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung. The Google file system. In *19th Symposium on Operating Systems Principles*, pages 29–43, Lake George, New York, 2003.

[9] S. Gorlatch. Systematic efficient parallelization of scan and other list homomorphisms. In L. Bougé, P. Fraigniaud, A. Mignotte, and Y. Robert, editors, *Euro-Par'96. Parallel Processing*, Lecture Notes in Computer Science 1124, pages 401–408. Springer-Verlag, 1996.

[10] Jim Gray. Sort benchmark home page. http://research.microsoft.com/barc/SortBenchmark/.

[11] William Gropp, Ewing Lusk, and Anthony Skjellum. *Using MPI: Portable Parallel Programming with the Message Passing Interface*. MIT Press, Cambridge, MA, 1999.

[12] L. Huston, R. Sukthankar, R. Wickremesinghe, M. Satyanarayanan, G. R. Ganger, E. Riedel, and A. Ailamaki. Diamond: A storage architecture for early discard in interactive search. In *Proceedings of the 2004 USENIX File and Storage Technologies FAST Conference*, April 2004.

[13] Richard E. Ladner and Michael J. Fischer. Parallel prefix computation. *Journal of the ACM*, 27(4):831–838, 1980.

[14] Michael O. Rabin. Efficient dispersal of information for security, load balancing and fault tolerance. *Journal of the ACM*, 36(2):335–348, 1989.

[15] Erik Riedel, Christos Faloutsos, Garth A. Gibson, and David Nagle. Active disks for large-scale data processing. *IEEE Computer*, pages 68–74, June 2001.

[16] Douglas Thain, Todd Tannenbaum, and Miron Livny. Distributed computing in practice: The Condor experience. *Concurrency and Computation: Practice and Experience*, 2004.

[17] Leslie G. Valiant. A bridging model for parallel computation. *Communications of the ACM*, 33(8):103–111, 1990.

[18] Jim Wyllie. SPsort: How to sort a terabyte quickly. http://alme1.almaden.ibm.com/cs/spsort.pdf.

---

## 附录 A：词频统计（Word Frequency）

本节包含一个程序，用于统计命令行指定的一组输入文件中每个唯一单词的出现次数。

```cpp
#include "mapreduce/mapreduce.h"

// User's map function
class WordCounter : public Mapper {
 public:
  virtual void Map(const MapInput& input) {
    const string& text = input.value();
    const int n = text.size();
    for (int i = 0; i < n; ) {
      // Skip past leading whitespace
      while ((i < n) && isspace(text[i]))
        i++;
      // Find word end
      int start = i;
      while ((i < n) && !isspace(text[i]))
        i++;
      if (start < i)
        Emit(text.substr(start,i-start),"1");
    }
  }
};
REGISTER_MAPPER(WordCounter);

// User's reduce function
class Adder : public Reducer {
 public:
  virtual void Reduce(ReduceInput* input) {
    // Iterate over all entries with the
    // same key and add the values
    int64 value = 0;
    while (!input->done()) {
      value += StringToInt(input->value());
      input->NextValue();
    }
    // Emit sum for input->key()
    Emit(IntToString(value));
  }
};
REGISTER_REDUCER(Adder);

int main(int argc, char** argv) {
  ParseCommandLineFlags(argc, argv);

  MapReduceSpecification spec;

  // Store list of input files into "spec"
  for (int i = 1; i < argc; i++) {
    MapReduceInput* input = spec.add_input();
    input->set_format("text");
    input->set_filepattern(argv[i]);
    input->set_mapper_class("WordCounter");
  }

  // Specify the output files:
  //   /gfs/test/freq-00000-of-00100
  //   /gfs/test/freq-00001-of-00100
  //   ...
  MapReduceOutput* out = spec.output();
  out->set_filebase("/gfs/test/freq");
  out->set_num_tasks(100);
  out->set_format("text");
  out->set_reducer_class("Adder");

  // Optional: do partial sums within map
  // tasks to save network bandwidth
  out->set_combiner_class("Adder");

  // Tuning parameters: use at most 2000
  // machines and 100 MB of memory per task
  spec.set_machines(2000);
  spec.set_map_megabytes(100);
  spec.set_reduce_megabytes(100);

  // Now run it
  MapReduceResult result;
  if (!MapReduce(spec, &result)) abort();

  // Done: 'result' structure contains info
  // about counters, time taken, number of
  // machines used, etc.

  return 0;
}
```

<!-- obsidian-publish-source: Operating System 操作系统/Pappers 论文/OSDI/MapReduce 大规模数据集上的简化数据处理.md -->