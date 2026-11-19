# 术语表（Glossary）

> **摘要**：收录 RoCEv2 学习、实现与运维中最常用的约 250 个术语——InfiniBand 规范术语、以太网/IP 术语与业界术语，给出中英对照、1–2 句解释和知识库详解链接。IB 术语以 Vol1 第 2 章 Glossary 为准（意译），其在 RoCEv2 下的特殊含义以 Annex A17 为准。
>
> **规范出处**：Vol1 第 2 章 Glossary（p.84–100）；Vol1 Annex A17（p.1928–1949）；Vol1 Annex A16（p.1911–1927）；Vol1 Annex A10 Glossary（p.1797–1799）；Vol2 第 2 章 Glossary（p.40–61）
>
> **前置阅读**：无（随查随用）；建议配合 [RDMA 基础](../00-overview/01-rdma-fundamentals.md) 与 [IBA 分层映射](../00-overview/03-architecture-and-layering.md)
>
> **关键词**：术语, 缩写, 中英对照, Glossary, IB, RoCEv2, DCB

## 1. 使用说明

- **排序**：按英文术语字母顺序（忽略大小写与符号），数字开头的术语放在最前；每个字母一节。
- **来源标记**（写在"解释"列开头）：
  - 〔IB〕Vol1 正文/术语表定义的 InfiniBand 术语；〔A17〕〔A16〕〔A10〕〔A11〕指对应附录；〔IB Vol2〕指物理层规范。
  - 〔外部: …〕IEEE / IETF / PCI-SIG 等其他标准；〔论文: …〕学术论文；〔业界〕厂商或开源实现中的通用说法（以所用版本文档为准）。
- **RoCEv2 适用性**：解释中写明"RoCEv2 不适用"的术语，仅在阅读 IB 规范或对比 IB 时需要理解。
- 同一概念的缩写与全称合并为一条；"详解"列链接到负责该主题的知识库文件。

## 2. 0–9

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| 8b/10b | 8b/10b 编码 | 〔IB Vol2〕IB SDR/DDR/QDR 的线路编码，10 bit 码组承载 8 bit 数据，提供直流平衡和码组同步，效率 80%〔Vol2 §5.2, p.87〕。 | [IB 物理层对比](04-ib-physical-layer-vs-ethernet.md) |
| 64b/66b | 64b/66b 编码 | 〔IB Vol2/外部: IEEE 802.3〕FDR/EDR/HDR 及 10G 以上以太网的块编码：2 bit 同步头（01 数据 / 10 控制）+ 64 bit 加扰负载〔Vol2 §5.3, p.100〕。 | [IB 物理层对比](04-ib-physical-layer-vs-ethernet.md) |
| 802.1Q | VLAN 标签标准 | 〔外部: IEEE 802.1Q〕定义 4 字节 VLAN 标签（TPID 0x8100 + PCP/DEI/VID）与优先级；RoCEv2 二层优先级（PCP）依赖它。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| 802.1Qaz | ETS / DCBX 标准 | 〔外部: IEEE 802.1Qaz〕定义增强传输选择（ETS）与 DCB 能力交换（DCBX）；规范以它作为 RoCEv2 QoS 的推荐手段〔Vol1 §A17.9.2, p.1946〕。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| 802.1Qbb | PFC 标准 | 〔外部: IEEE 802.1Qbb〕按优先级的链路级流控，规范称其"非常适合 RoCEv2 流量"〔Vol1 §A17.9.1, p.1946〕。 | [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md) |
| 802.3x | 以太网 PAUSE | 〔外部: IEEE 802.3〕整链路级暂停帧，不区分优先级，会阻塞所有流量，RoCE 部署中一般被 PFC 取代。 | [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md) |
| 4791 | RoCEv2 UDP 目的端口 | 〔A17/外部: IANA〕IANA 为 RoCEv2 分配的 UDP 目的端口。规范正文只写"IANA 分配值"（CA17-20），脚注注明分配后更新〔Vol1 §A17.3.2.2, p.1934–1935〕。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |

## 3. A

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| A (AckReq) | 确认请求位 | 〔IB〕BTH 字节 8（从 0 起算）的最高位，置 1 时请求响应方为该包调度确认〔Vol1 §9.2.12, p.261〕。 | [BTH](../01-packet-format/03-bth.md) |
| ACK (Acknowledge) | 确认（正确认） | 〔IB〕响应方回送的确认包，RC 下 OpCode 为 Acknowledge（0x11），携带 AETH（Syndrome + MSN）。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |
| Acknowledge Coalescing | 确认合并 | 〔IB〕响应方用一个 ACK（PSN 为最近完成的请求）同时确认此前多个请求〔Vol1 §9.7.5.1.2, p.332〕。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |
| Active / Passive | 主动方 / 被动方 | 〔IB〕连接建立中发起请求的一方 / 等待请求的一方，类似 TCP 的 CONNECT / LISTEN。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| AETH (ACK Extended Transport Header) | ACK 扩展传输头 | 〔IB〕4 字节：Syndrome(8) + MSN(24)；出现在 ACK、Atomic ACK 及 READ 响应 First/Last/Only 包中〔Vol1 §9.3.5, p.265〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| AH (Address Handle) | 地址句柄 | 〔IB〕封装 Address Vector 的对象，UD 发送时在 WR 中引用以指定目的端口。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| AM (Attribute Modifier) | 属性修饰符 | 〔IB〕MAD 头中 32 bit 字段，进一步限定被操作属性的范围。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| APM (Automatic Path Migration) | 自动路径迁移 | 〔IB〕CA 以 QP 为单位切换到预设备用路径，用 BTH 的 MigReq 位通知对端；RoCEv2 中 AV 的备用路径分量按 §A17.4.2 修改。 | [QP 状态机](../03-verbs-and-resources/02-qp-state-machine.md) |
| APR (Alternate Path Response) | 备用路径响应 | 〔IB CM〕对 LAP 的应答；RoCEv2 端口间不得发送状态码 7"Proposed Alternate Remote Port LID rejected"（CA17-37）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| ARP (Address Resolution Protocol) | 地址解析协议 | 〔外部: RFC 826〕IPv4 地址到 MAC 的解析。RoCEv2 的 L3→L2 解析不在规范范围内，通常复用 OS 的 ARP/ND〔Vol1 §A17.4.2, p.1937〕。 | [寻址](../00-overview/04-addressing.md) |
| Asynchronous Error / Event | 异步错误 / 异步事件 | 〔IB〕无法通过立即返回或完成状态报告的错误/事件，可关联到 CQ、端口或 QP，也可为非关联事件。 | [错误处理](../02-transport/08-error-handling.md) |
| AtomicAckETH | 原子确认扩展传输头 | 〔IB〕8 字节，携带原子操作执行前远端内存的原始值（OrigRemDt）〔Vol1 §9.3.5.3, p.266〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| AtomicETH | 原子扩展传输头 | 〔IB〕28 字节：VA(64) + R_Key(32) + Swap/Add 数据(64) + Compare 数据(64)〔Vol1 §9.3.4, p.265〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| Atomic Operation | 原子操作 | 〔IB〕对远端 64 bit、8 字节自然对齐地址执行的 CmpSwap 或 FetchAdd，返回原值；未对齐时返回 Invalid Request NAK〔Vol1 §9.4.5, p.283〕。 | [操作类型](../02-transport/02-operations.md) |
| Attribute (MAD) | 属性 | 〔IB〕MAD 承载的管理数据对象，由 AttributeID 标识，如 CM 的 ConnectRequest = 0x0010。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| AV (Address Vector) | 地址向量 | 〔IB/A17〕描述远端端口和路径参数的集合。RoCEv2 中含 DGID、SGID index、Hop Limit、Traffic Class、Flow Label、SL、目的 L2 地址等，"发送 GRH"标志恒为 TRUE〔Vol1 §A17.4.2, p.1937〕。 | [寻址](../00-overview/04-addressing.md) |

## 4. B

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| Base LID | 基础 LID | 〔IB〕端口数值最小的 LID（Path Bits 为 0）；RoCEv2 端口的 Query HCA 中 Base LID/LMC 无效并被忽略〔Vol1 §A17.5.1, p.1940〕。 | [寻址](../00-overview/04-addressing.md) |
| BDP (Bandwidth-Delay Product) | 带宽时延积 | 〔通用〕链路速率 × 往返时延，决定维持满速所需的在途数据量与缓冲需求。 | [常数与公式](05-constants-and-formulas.md) |
| BECN (Backward Explicit Congestion Notification) | 后向显式拥塞通知 | 〔A10〕IB 拥塞控制中由目的端在 ACK/CNP 的 BTH B 位回告拥塞；RoCEv2 端口不使用该位且接收时忽略（CA17-29）。 | [IB CC 对比](../05-congestion-control-and-qos/07-ib-cc-vs-rocev2-cc.md) |
| Binding (MW Bind) | 绑定 | 〔IB〕把已注册 MR 中的一段虚拟地址范围与 Memory Window 关联并赋予访问权限的操作。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| B_Key (Baseboard Management Key) | 基板管理密钥 | 〔IB〕基板管理类 MAD 的认证密钥；RoCEv2 不适用（基板管理类不适用，§A17.6）。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| BM (Baseboard Management) | 基板管理 | 〔IB〕MgmtClass 0x05，经 IB 链路隧道传输 IB-ML 命令；RoCEv2 不适用（§A17.6）。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| BTH (Base Transport Header) | 基本传输头 | 〔IB〕所有 IB 传输报文都有的 12 字节头：OpCode、SE、M、PadCnt、TVer、P_Key、F/B、DestQP、A、PSN〔Vol1 §9.2, p.256–262〕。 | [BTH](../01-packet-format/03-bth.md) |

## 5. C

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| CA (Channel Adapter) | 通道适配器 | 〔IB〕终结链路并执行传输层功能的设备，分为 HCA 与 TCA。 | [分层架构](../00-overview/03-architecture-and-layering.md) |
| CC (Congestion Control, IB) | IB 拥塞控制 | 〔A10〕基于交换机 FECN 标记、目的端 BECN 回告和 CCT 速率表的拥塞控制；对 RoCEv2 端口不适用〔Vol1 §A17.4.8, p.1940〕。 | [IB CC 对比](../05-congestion-control-and-qos/07-ib-cc-vs-rocev2-cc.md) |
| CCT (Congestion Control Table) | 拥塞控制表 | 〔A10〕IB CC 中按 CCTI 索引的注入速率延迟（IPD）表；RoCEv2 不使用。 | [IB CC 对比](../05-congestion-control-and-qos/07-ib-cc-vs-rocev2-cc.md) |
| CE (Congestion Experienced) | 拥塞经历 | 〔外部: RFC 3168〕IP ECN 字段取值 11，由交换机在拥塞时标记；支持 RCM 的接收端据此生成 CNP（CA17-44）。 | [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md) |
| CI (Channel Interface) | 通道接口 | 〔IB〕由 HCA 硬件、固件和驱动共同呈现给 Verbs 使用者的通道实现。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| CM (Communication Manager / Management) | 通信管理器 / 通信管理 | 〔IB〕负责连接建立、释放与备用路径管理的协议和实体（第 12 章）。RoCEv2 沿用 IB CM，CM 报文就是发往 QP1 的普通 RoCEv2 报文〔Vol1 §A17.6.1, p.1943〕。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| CmpSwap (Compare & Swap) | 比较并交换 | 〔IB〕原子操作：远端 64 bit 值等于 Compare 数据时写入 Swap 数据，总是返回原值。 | [操作类型](../02-transport/02-operations.md) |
| CNP (Congestion Notification Packet) | 拥塞通知报文 | 〔A17/A10〕接收端发回源端的拥塞通知。RoCEv2 CNP 的 BTH OpCode = 0b1000_0001（0x81），后接 16 字节保留区〔Vol1 Figure 349, p.1948〕；IB CC 的 CNP 为 0x80。 | [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md) |
| Completion Error | 完成错误 | 〔IB〕通过完成状态（CQE）报告的永久性接口错误或处理错误。 | [错误处理](../02-transport/08-error-handling.md) |
| Completion Event Handler | 完成事件处理函数 | 〔IB〕消费者请求完成通知后，CQ 中新增条目时被调用的处理函数。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| CP (Congestion Point) | 拥塞点 | 〔论文: DCQCN, SIGCOMM 2015〕DCQCN 中执行 ECN 标记的交换机出口队列。 | [DCQCN](../05-congestion-control-and-qos/04-dcqcn.md) |
| CQ (Completion Queue) | 完成队列 | 〔IB〕存放 CQE 的队列，可被多个 QP 的工作队列共享。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| CQE (Completion Queue Entry) | 完成队列项 | 〔IB〕Work Completion 在 CI 内部的表示，俗称"cookie"。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| CRC (Cyclic Redundancy Check) | 循环冗余校验 | 〔通用〕IB 报文带 ICRC（32 bit）和 VCRC（16 bit）；RoCE/RoCEv2 用以太网 FCS 代替 VCRC。 | [ICRC](../01-packet-format/06-icrc.md) |
| Credit (End-to-End Credit) | 端到端信用 | 〔IB〕RC 响应方在 AETH Syndrome 低 5 bit 编码的"可用接收 WQE 数（相对 MSN）"，编码 11111 表示信用无效〔Vol1 §9.7.5.2.1, p.349〕。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |

## 6. D

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| Data Segment | 数据段 | 〔IB〕WR 中描述一段虚拟连续缓冲的三元组（VA、L_Key/R_Key、长度），Verbs 编程中常称 SGE。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| DCB (Data Center Bridging) | 数据中心桥接 | 〔外部: IEEE 802.1〕PFC、ETS、DCBX、QCN 等以太网增强功能的统称。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| DCBX (DCB Capability Exchange) | DCB 能力交换协议 | 〔外部: IEEE 802.1Qaz〕基于 LLDP 在链路两端交换/协商 PFC、ETS 与应用优先级配置。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| DCQCN (Data Center QCN) | 数据中心量化拥塞通知 | 〔论文: DCQCN, SIGCOMM 2015〕基于 ECN + CNP 的速率型拥塞控制算法，是 RCM 框架下最常见的实现，但**不是**规范内容。 | [DCQCN](../05-congestion-control-and-qos/04-dcqcn.md) |
| DestQP | 目的 QP | 〔IB〕BTH 中 24 bit 目的 QP 号；DestQP = 0 的 RoCEv2 报文须静默丢弃（CA17-33），0xFFFFFF 用于 UD 多播。 | [BTH](../01-packet-format/03-bth.md) |
| DETH (Datagram Extended Transport Header) | 数据报扩展传输头 | 〔IB〕8 字节：Q_Key(32) + 保留(8) + SrcQP(24)，用于 UD 与 RD〔Vol1 §9.3.2, p.263〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| DGID (Destination GID) | 目的 GID | 〔IB/A17〕目的端口 GID；RoCEv2 中即目的 IP：IPv6 原值或 ::ffff:IPv4〔Vol1 §A17.4.1.1, p.1936〕。 | [寻址](../00-overview/04-addressing.md) |
| DLID (Destination LID) | 目的 LID | 〔IB/A17〕LRH 中的目的本地标识。RoCEv2 中 AV 的 DLID 分量泛化为"目的 L2 地址"（DMAC），取 0xFFFFFFFFFFFF 表示由 Verbs 实现自行解析〔Vol1 §A17.4.2, p.1937〕。 | [寻址](../00-overview/04-addressing.md) |
| DMA (Direct Memory Access) | 直接内存访问 | 〔通用〕设备不经 CPU 直接读写主机内存；RNIC 通过 DMA 读取 WQE/数据并写回数据与 CQE。 | [RNIC 架构](../06-hardware-implementation/01-rnic-architecture.md) |
| DMAC / SMAC | 目的 / 源 MAC 地址 | 〔外部: IEEE 802.3〕以太网头中的 48 bit 地址；RoCEv2 的 SMAC 由所选 SGID 隐含，DMAC 由 L3→L2 解析得到或经 AV 传入〔Vol1 §A17.4.2, p.1937–1938〕。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |
| DMALen (DMA Length) | DMA 长度 | 〔IB〕RETH 中 32 bit，RDMA 操作总字节数，HCA 取值 0～2^31（C9-9）。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| Doorbell | 门铃 | 〔业界〕软件写 RNIC 的 MMIO 寄存器，通知有新 WQE 待处理（或 CQ 已被消费）的机制，具体格式厂商相关。 | [RNIC 架构](../06-hardware-implementation/01-rnic-architecture.md) |
| DREQ / DREP | 断开请求 / 断开应答 | 〔IB CM〕连接释放的消息对（AttributeID 0x0015 / 0x0016）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| DSCP (Differentiated Services Code Point) | 区分服务码点 | 〔外部: RFC 2474〕IPv4 TOS / IPv6 Traffic Class 的高 6 bit；RoCEv2 发送时取自 AV 的 Traffic Class（CA17-4、CA17-13）。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| Duplicate Request | 重复请求 | 〔IB〕PSN 落在响应方重复区的请求：SEND/WRITE 不重执行只回 ACK（C9-105），READ 须重新生成读数据（C9-107），原子操作返回保存的结果（o9-66）。 | [响应方](../02-transport/07-responder.md) |

## 7. E

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| ECMP (Equal-Cost Multi-Path) | 等价多路径 | 〔外部〕路由器对报文字段做哈希在多条等价路径间分流；RoCEv2 的 UDP 源端口作为流熵参与哈希〔Vol1 §A17.9.4, p.1948〕。 | [ECMP](../05-congestion-control-and-qos/06-ecmp-and-load-balancing.md) |
| ECN (Explicit Congestion Notification) | 显式拥塞通知 | 〔外部: RFC 3168〕IP 头 2 bit：00 Not-ECT、01 ECT(1)、10 ECT(0)、11 CE；是 RoCEv2 拥塞管理（RCM）的信令基础。 | [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md) |
| ECT (ECN-Capable Transport) | ECN 能力传输 | 〔外部: RFC 3168〕ECN = 01 或 10，表示报文可被网络标记。支持 RCM 的 HCA 可发送 ECT，不支持者须置 00（CA17-5、CA17-14）。 | [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md) |
| EEC / EE Context (End-to-End Context) | 端到端上下文 | 〔IB〕RD 服务的端点，承载可靠传输状态，由 EECN 标识。 | [服务类型](../02-transport/01-service-types.md) |
| End-to-End Flow Control | 端到端流控 | 〔IB〕防止在对端未投递接收缓冲时发送消息的机制；RC 中由 AETH 信用与 MSN 得出 LSN〔Vol1 §9.7.7.2, p.371〕。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |
| ePSN (expected PSN) | 期望 PSN | 〔IB〕响应方期望下一个新请求携带的 PSN；等于 ePSN 为新请求，落在有效区其余部分为重复请求〔Vol1 §9.7.1, p.309〕。 | [响应方](../02-transport/07-responder.md) |
| ETH (Extended Transport Header) | 扩展传输头 | 〔IB〕BTH 之后按 OpCode 出现的头：RDETH、DETH、XRCETH、RETH、AtomicETH、AETH、AtomicAckETH、ImmDt、IETH。注意不要与以太网（Ethernet）缩写混淆。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| Ethertype | 以太类型 | 〔外部: IEEE 802.3〕以太网头中标识上层协议的 16 bit：IPv4 0x0800、IPv6 0x86DD、RoCE v1 0x8915、VLAN 0x8100、MAC 控制（PAUSE/PFC）0x8808。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |
| ETS (Enhanced Transmission Selection) | 增强传输选择 | 〔外部: IEEE 802.1Qaz〕按流量类别分配带宽的调度机制；规范推荐 RoCEv2 QoS 使用〔Vol1 §A17.9.2, p.1946〕。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| EUI-64 | 64 位扩展唯一标识 | 〔外部: IEEE〕IEEE 定义的 64 bit 标识；IB 的 GUID 即 EUI-64 格式〔Vol1 §4.1, p.159〕。 | [寻址](../00-overview/04-addressing.md) |

## 8. F

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| Fabric | 网络结构 / 交换网络 | 〔IB〕连接一组 CA 的链路、交换机和路由器的集合。 | [分层架构](../00-overview/03-architecture-and-layering.md) |
| Fast Register (FRMR / Fast Register PMR) | 快速注册 | 〔IB〕通过 Post Send WR 在已有 L_Key（及 R_Key）上完成内存注册，避免慢速 Verbs 调用。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| FCS (Frame Check Sequence) | 帧校验序列 | 〔外部: IEEE 802.3〕以太网帧尾 4 字节 CRC-32，逐跳校验/重算；RoCE 以 FCS 代替 IB 的 VCRC（CA16-3）。 | [ICRC](../01-packet-format/06-icrc.md) |
| FEC (Forward Error Correction) | 前向纠错 | 〔IB Vol2/外部: IEEE 802.3〕物理层纠错编码：IB 定义 Fire-code (2112,2080) 与 RS(528,514)、RS(544,514)、RS(271,257)〔Vol2 §5.5, p.110〕；以太网 100G 以上普遍用 RS-FEC。 | [IB 物理层对比](04-ib-physical-layer-vs-ethernet.md) |
| FECN (Forward Explicit Congestion Notification) | 前向显式拥塞通知 | 〔A10〕IB 交换机在拥塞时置 BTH 的 F 位。RoCEv2 不使用（CA17-29），改由 IP ECN = CE 表达同一语义〔Vol1 §A17.9.3, p.1947〕。 | [IB CC 对比](../05-congestion-control-and-qos/07-ib-cc-vs-rocev2-cc.md) |
| Fence (Fence Indicator) | 栅栏 | 〔IB〕WR 属性：置位时该 WR 须等同一 SQ 上之前的 RDMA READ 与 Atomic 全部完成后才开始处理（C10-100）。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| FetchAdd | 取并加 | 〔IB〕原子操作：把 Add 数据加到远端 64 bit 值上并返回原值。 | [操作类型](../02-transport/02-operations.md) |
| Flow Label | 流标签 | 〔外部: RFC 8200 / IB GRH〕IPv6 头 20 bit 字段，可被沿途改写，ICRC 计算时以 1 替换（CA17-22）；Linux 用它派生 RoCEv2 UDP 源端口〔业界: Linux rdma_flow_label_to_udp_sport()〕。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |
| Flush (WR Flush Error) | 冲刷 | 〔IB〕QP 进入 Error 态后，未完成及新投递的 WR 以"冲刷错误"状态完成，不再执行。 | [错误处理](../02-transport/08-error-handling.md) |

## 9. G

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| GID (Global Identifier) | 全局标识符 | 〔IB/A17〕128 bit 单播/多播标识。RoCEv2 的 L3 地址统称 GID：IPv6 地址直接作 GID，IPv4 映射为 ::ffff:a.b.c.d〔Vol1 §A17.4.1.1, p.1936〕。 | [寻址](../00-overview/04-addressing.md) |
| GID Index (SGID Index) | GID 索引 | 〔IB/A17〕AV 中引用端口 GID 表项的索引；源 IP、源 MAC 与出站协议都由该表项决定（CA17-11、CA17-18）。 | [寻址](../00-overview/04-addressing.md) |
| GID Table (Port GID Table) | 端口 GID 表 | 〔IB/A17〕端口上配置的全部 L3 地址；RoCEv2 中由软件栈随 IP 地址增删维护〔Vol1 §A17.4.3, p.1938〕。 | [寻址](../00-overview/04-addressing.md) |
| GID Type | GID 类型 | 〔A17〕GID 表项属性：IPv4、IPv6 或 IB GID（后者用于与 RoCE v1 互通），决定出站报文协议（CA17-26、§A17.8）。 | [寻址](../00-overview/04-addressing.md) |
| GMP (General Management Packet) | 通用管理包 | 〔IB〕经 GSI（QP1）收发的 MAD，与经 QP0 的 SMP 相对；CM 消息属于 GMP。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| Go-back-N | 回退 N 重传 | 〔通用〕从出错/超时的 PSN 起重传其后全部请求；IB RC 请求方在 NAK 或超时后按此方式恢复。 | [请求方](../02-transport/06-requester.md) |
| GPUDirect RDMA | GPU 直连 RDMA | 〔业界: NVIDIA〕允许 RNIC 直接 DMA 读写 GPU 显存，是 AI 训练集群的关键能力（厂商相关，以官方文档为准）。 | [典型场景](../07-engineering-practice/06-use-cases.md) |
| GRH (Global Route Header) | 全局路由头 | 〔IB〕40 字节、格式与 IPv6 头相同的 IB 网络层头（NxtHdr 0x1B 表示 IBA 传输）；RoCE v1 每包必带，RoCEv2 以 IP 头替代。 | [RoCE 演进](../00-overview/02-roce-evolution.md) |
| GSI (General Services Interface) | 通用服务接口 | 〔IB〕子网管理以外的管理服务接口（CM、性能管理等），使用 QP1；RoCEv2 中 Get Special QP 仅适用于 GSI〔Vol1 §A17.5.8, p.1942〕。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| GUID (Globally Unique Identifier) | 全局唯一标识 | 〔IB〕厂商分配的 EUI-64 标识（Node GUID、Port GUID），IB 中与 GID 前缀组成默认 GID。 | [寻址](../00-overview/04-addressing.md) |

## 10. H

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| HCA (Host Channel Adapter) | 主机通道适配器 | 〔IB〕支持 Verbs 接口的 CA；RoCEv2 网卡在规范语境中即 RoCEv2 HCA（业界称 RNIC）。 | [RNIC 架构](../06-hardware-implementation/01-rnic-architecture.md) |
| Headroom (PFC Headroom) | PFC 余量缓冲 | 〔业界〕端口发出 PFC 暂停后仍会到达的在途数据所需的缓冲，取决于线缆长度、速率、MTU 与响应时延。 | [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md) |
| Hop Limit | 跳数限制 | 〔外部: RFC 8200 / IB〕IPv6 与 GRH 的跳数字段；RoCEv2 中 IPv4 TTL 与 IPv6 Hop Limit 都取自 AV 的 Hop Limit（CA17-9、CA17-17），ICRC 中屏蔽。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |
| HPCC (High Precision Congestion Control) | 高精度拥塞控制 | 〔论文: HPCC, SIGCOMM 2019〕利用交换机带内遥测（INT）精确调整发送速率的拥塞控制。 | [高级拥塞控制](../05-congestion-control-and-qos/05-advanced-cc-and-lossy-roce.md) |

## 11. I

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| IBA (InfiniBand Architecture) | InfiniBand 体系结构 | 〔IB〕IBTA 定义的物理、链路、网络、传输四层加管理框架与 Verbs 的完整体系。 | [分层架构](../00-overview/03-architecture-and-layering.md) |
| IBTA (InfiniBand Trade Association) | InfiniBand 行业协会 | 〔外部〕发布 IB 规范（含 RoCE/RoCEv2 附录）并负责 RoCE 架构推广的组织〔Vol2 §A3.1, p.688〕。 | [RoCE 演进](../00-overview/02-roce-evolution.md) |
| ICRC (Invariant CRC) | 不变 CRC | 〔IB/A17〕覆盖端到端不变字段的 32 bit CRC。RoCEv2 以 64 bit 全 1 起算，覆盖 IP 头到负载，可变字段以 1 替换（CA17-22）。 | [ICRC](../01-packet-format/06-icrc.md) |
| IETH (Invalidate Extended Transport Header) | 失效扩展传输头 | 〔IB〕4 字节 R_Key，用于 SEND with Invalidate，响应方据此使对应 MR/MW 失效〔Vol1 §9.3.7, p.266〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| IHL (Internet Header Length) | IPv4 头长度 | 〔外部: RFC 791〕IPv4 头长度（以 4 字节为单位）；RoCEv2 规定为 5，即不带选项（CA17-3）。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |
| ImmDt (Immediate Data) | 立即数 | 〔IB〕4 字节，随 SEND / RDMA WRITE with Immediate 发送并出现在接收端 CQE 中〔Vol1 §9.3.6, p.266〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| Initiator Depth / Responder Resources | 发起深度 / 响应资源 | 〔IB CM〕REQ/REP 字段：本端将发起 / 可接受的未完成 RDMA READ 与 Atomic 数量〔Vol1 §12.7.29–12.7.30, p.736〕。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| Inline Data | 内联数据 | 〔业界: libibverbs IBV_SEND_INLINE〕小消息直接拷贝进 WQE，省去 RNIC 一次 DMA 读取（上限厂商相关）。 | [性能调优](../07-engineering-practice/03-performance-tuning.md) |
| Invalidate (Local / Remote) | 失效操作 | 〔IB〕使 L_Key/R_Key 失去访问能力但保留资源以便复用；本地经 SQ 执行，远端经 SEND with Invalidate 触发。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| IP CM Service (RDMA IP CM Service) | RDMA IP 连接管理服务 | 〔A11〕在 CM REQ 私有数据中携带 IP 地址与端口，并把 IP 端口映射到 Service ID（前缀 0x0000000001）；rdma_cm 使用此格式。 | [IP CM 与 rdma_cm](../04-connection-management/02-ip-cm-and-rdma-cm.md) |
| IPD (Inter-Packet Delay) | 包间延迟 | 〔IB〕静态速率控制参数，限制源端注入速率以适配较慢的路径〔Vol1 §9.11, p.456〕。 | [请求方](../02-transport/06-requester.md) |
| IRD / ORD | 入向 / 出向读原子深度 | 〔业界〕QP 可同时接受 / 发起的未完成 RDMA READ 与 Atomic 数，对应 CM 的 Responder Resources / Initiator Depth。 | [请求方](../02-transport/06-requester.md) |
| IRN (Improved RoCE NIC) | 改进型 RoCE 网卡 | 〔论文: IRN, SIGCOMM 2018〕主张以选择性重传与 BDP 限流取代 PFC 的有损 RoCE 方案。 | [高级拥塞控制](../05-congestion-control-and-qos/05-advanced-cc-and-lossy-roce.md) |
| iSER (iSCSI Extensions for RDMA) | iSCSI RDMA 扩展 | 〔外部: RFC 7145 / A12〕基于 RDMA 的 iSCSI 数据传输。 | [典型场景](../07-engineering-practice/06-use-cases.md) |
| iWARP | iWARP | 〔外部: RFC 5040/5041/5044〕基于 TCP 的 RDMA 协议族（RDMAP/DDP/MPA），与 RoCE 是不同技术路线。 | [RDMA 基础](../00-overview/01-rdma-fundamentals.md) |

## 12. K

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| Key | 密钥 | 〔IB〕限制资源访问的类口令结构：L_Key、R_Key、P_Key、Q_Key、M_Key、B_Key。 | [分层架构](../00-overview/03-architecture-and-layering.md) |

## 13. L

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| LAP (Load Alternate Path) | 加载备用路径 | 〔IB CM〕为已建立连接加载备用路径；RoCEv2 中其 Alternate Local/Remote Port LID 保留并在接收时忽略（CA17-36）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| LID (Local Identifier) | 本地标识符 | 〔IB〕SM 分配的 16 bit 子网内地址；RoCEv2 中 LRH 及其字段全部不适用〔Vol1 §A17.4.1.2, p.1936〕。 | [寻址](../00-overview/04-addressing.md) |
| L_Key (Local Key) | 本地密钥 | 〔IB〕注册内存时由 Verbs 生成，与 VA 一起授权 HCA 访问本地内存，在 WR 的数据段中使用。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| LLDP (Link Layer Discovery Protocol) | 链路层发现协议 | 〔外部: IEEE 802.1AB〕以太网邻居发现协议（Ethertype 0x88CC），DCBX 的载体。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| LMC (LID Mask Control) | LID 掩码控制 | 〔IB〕SM 为端口配置的 Path Bits 位数（0–7）；RoCEv2 不适用。 | [寻址](../00-overview/04-addressing.md) |
| Local ACK Timeout | 本地 ACK 超时 | 〔IB〕QP 属性，5 bit 指数：Ttr = 4.096 µs × 2^值，0 表示禁用定时器（C9-140）。 | [请求方](../02-transport/06-requester.md) |
| Loopback | 回环 | 〔IB〕自寻址报文在 CA 内部回送、不上线路（C17-18）；Loopback GID ::1 只用于 Raw IPv6，不得出现在 IBA 报文中〔Vol1 §4.1.1, p.161〕。 | [RNIC 架构](../06-hardware-implementation/01-rnic-architecture.md) |
| Lossless Ethernet | 无损以太网 | 〔A17/业界〕通过 PFC 等链路级流控避免拥塞丢包的以太网；规范建议 RoCEv2 网络配置为无损，但"无损"不等于绝不丢包〔Vol1 §A17.9.1, p.1946〕。 | [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md) |
| Lossy RoCE | 有损 RoCE | 〔业界〕不依赖 PFC、靠拥塞控制和改进的重传在可丢包网络上运行 RoCE 的方案（厂商相关）。 | [高级拥塞控制](../05-congestion-control-and-qos/05-advanced-cc-and-lossy-roce.md) |
| LRH (Local Route Header) | 本地路由头 | 〔IB〕8 字节 IB 链路层头（VL、LVer、SL、LNH、DLID、PktLen、SLID）；RoCE/RoCEv2 中由以太网 MAC 头替代。 | [RoCE 演进](../00-overview/02-roce-evolution.md) |
| LSN (Limit Sequence Number) | 限制序列号 | 〔IB〕MSN 加上信用值；请求方可以发送 SSN ≤ LSN 的消耗接收 WQE 的请求〔Vol1 §9.7.7.2, p.379〕。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |

## 14. M

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| M (MigReq) | 迁移请求位 | 〔IB〕BTH 中 1 bit，用于通告 APM 迁移状态；RoCEv2 CNP 中须置 0〔Vol1 Figure 349, p.1948〕。 | [BTH](../01-packet-format/03-bth.md) |
| MAD (Management Datagram) | 管理数据报 | 〔IB〕负载固定 256 字节（24 字节公共头 + 232 字节数据）的管理报文（C13-3），经 UD 服务收发。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| Message | 消息 | 〔IB〕CA 之间由一个或多个包组成的一次信息传送，通常对应一个 WR。 | [操作类型](../02-transport/02-operations.md) |
| MGID / MLID | 多播 GID / 多播 LID | 〔IB〕多播组标识；RoCEv2 中 MLID 无效，Attach/Detach 多播时被忽略〔Vol1 §A17.5.6, p.1942〕。 | [不可靠服务](../02-transport/04-unreliable-services.md) |
| MgmtClass | 管理类 | 〔IB〕MAD 头中 8 bit 管理类别字段，CM 为 0x07〔Vol1 Table 122, p.784〕。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| M_Key (Management Key) | 管理密钥 | 〔IB〕SMP 中认证管理者的密钥；RoCEv2 无子网管理，不适用。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| MPT / MTT | 内存保护表 / 地址翻译表 | 〔业界〕RNIC 中存放 MR/MW 属性与 VA→PA 映射的硬件表结构（命名为业界习惯用语）。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| MR (Memory Region) | 内存区域 | 〔IB〕已注册、HCA 可本地访问并可选远程访问的虚拟连续内存区，注册后得到 L_Key/R_Key。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| MRA (Message Receipt Acknowledgment) | 消息接收确认 | 〔IB CM〕接收方无法在超时内正式应答时发送，携带 Service Timeout 以延长对端等待〔Vol1 §12.6.6, p.720〕。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| MSN (Message Sequence Number) | 消息序列号 | 〔IB〕AETH 中 24 bit，响应方最近完成的消息序号，用于请求方完成处理和信用计算。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |
| MTU (Maximum Transfer Unit) | 最大传输单元 | 〔IB/以太网〕IB 中指包负载上限（256/512/1024/2048/4096 字节）；以太网中指 IP 层最大包长。RoCE 路径 MTU 必须小于以太网 MTU 减去头部开销。 | [帧布局](../01-packet-format/01-frame-layout.md) |
| MTUCap / Active MTU / Neighbor MTU | MTU 能力 / 活动 MTU / 邻居 MTU | 〔IB〕端口支持的最大 MTU / 当前生效 MTU，编码 1=256、2=512、3=1024、4=2048、5=4096〔Vol1 PortInfo, p.906–909〕。 | [常数与公式](05-constants-and-formulas.md) |
| Multicast | 多播 | 〔IB〕一个包投递到多个端口；仅 UD 支持，DestQP 固定为 0xFFFFFF（o10-35）。 | [不可靠服务](../02-transport/04-unreliable-services.md) |
| MW (Memory Window) | 内存窗口 | 〔IB〕绑定到 MR 子区间、拥有独立 R_Key 的远程访问对象，分 Type 1、Type 2A、Type 2B。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |

## 15. N

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| NAK (Negative Acknowledge) | 否定确认 | 〔IB〕AETH Syndrome[6:5] = 11 的确认，低 5 bit 为 NAK 码：0 PSN 序列错、1 无效请求、2 远端访问错、3 远端操作错、4 无效 RD 请求〔Vol1 Table 47, p.349〕。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |
| NCCL | NVIDIA 集合通信库 | 〔业界: NVIDIA〕GPU 集合通信库，在 RoCEv2/IB 网络上通过 Verbs 实现 AllReduce 等操作。 | [典型场景](../07-engineering-practice/06-use-cases.md) |
| ND (Neighbor Discovery) | 邻居发现 | 〔外部: RFC 4861〕IPv6 的 L3→L2 解析机制，RoCEv2 IPv6 场景由 OS 提供。 | [寻址](../00-overview/04-addressing.md) |
| NP (Notification Point) | 通知点 | 〔论文: DCQCN〕DCQCN 中的接收端角色：检测 CE 标记并向源端发送 CNP。 | [DCQCN](../05-congestion-control-and-qos/04-dcqcn.md) |
| NVMe-oF (NVMe over Fabrics) | NVMe over Fabrics | 〔外部: NVM Express〕基于 RDMA（含 RoCEv2）、TCP 或 FC 的远程 NVMe 存储协议。 | [典型场景](../07-engineering-practice/06-use-cases.md) |

## 16. O

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| ODP (On-Demand Paging) | 按需分页 | 〔业界: Linux/mlx5〕无需钉住内存的注册方式，RNIC 访问缺页时由驱动与 OS 协作补齐映射（厂商相关）。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| OpCode (Operation Code) | 操作码 | 〔IB〕BTH 首字节：高 3 bit 为服务类型（000 RC、001 UC、010 RD、011 UD、100 CNP、101 XRC、110/111 厂商自定义），低 5 bit 为具体操作〔Vol1 Table 38, p.257–259〕。 | [OpCode](../01-packet-format/05-opcodes.md) |
| Outstanding | 未完成 / 在途 | 〔IB〕WR 已投递但完成尚未被取回；或报文已发出但尚未被确认。 | [请求方](../02-transport/06-requester.md) |

## 17. P

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| Packet | 包 / 报文 | 〔IB〕IB 数据传送与路由的不可分单位，由若干头、包负载和一或两个 CRC 组成。 | [帧布局](../01-packet-format/01-frame-layout.md) |
| PacketLifeTime | 包生存时间 | 〔IB〕报文可能在网络中存在的最长时间（PathRecord 字段，4.096 µs × 2^值），用于推导 ACK 超时与 TimeWait〔Vol1 §12.7.34, p.737〕。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| Packet Payload | 包负载 | 〔IB〕传输头之后、CRC 之前的部分，最多 4096 字节〔Vol1 Glossary, p.92〕。 | [帧布局](../01-packet-format/01-frame-layout.md) |
| PadCnt (Pad Count) | 填充计数 | 〔IB〕BTH 中 2 bit，表示负载末尾追加 0–3 个填充字节，使负载为 4 字节整数倍〔Vol1 §9.2.5, p.260〕。 | [BTH](../01-packet-format/03-bth.md) |
| PAM4 | 4 电平脉冲幅度调制 | 〔IB Vol2/外部: IEEE 802.3〕每符号 2 bit 的调制；IB HDR 用 26.5625 GBd PAM4（53.125 Gb/s/lane）并使用格雷码映射〔Vol2 §5.4, p.109〕。 | [IB 物理层对比](04-ib-physical-layer-vs-ethernet.md) |
| Partition | 分区 | 〔IB〕允许相互通信的端口集合，一个端口可同时属于多个分区。 | [寻址](../00-overview/04-addressing.md) |
| Path Bits | 路径位 | 〔IB〕LID 中可变的低位，用于在子网内选路；RoCEv2 中 AV 的 Source Path Bits 不适用〔Vol1 §A17.4.2, p.1937〕。 | [寻址](../00-overview/04-addressing.md) |
| PAUSE Quanta | 暂停量子 | 〔外部: IEEE 802.3〕PAUSE/PFC 暂停时间单位，1 quanta = 512 bit 时间（100 Gb/s 下为 5.12 ns）。 | [常数与公式](05-constants-and-formulas.md) |
| PCIe (PCI Express) | PCI Express 总线 | 〔外部: PCI-SIG〕主机与 RNIC 之间的 I/O 总线；其带宽、读往返时延常是 RNIC 性能瓶颈。 | [性能调优](../07-engineering-practice/03-performance-tuning.md) |
| PCP (Priority Code Point) | 优先级码点 | 〔外部: IEEE 802.1Q〕VLAN 标签中 3 bit 优先级；RoCEv2 中 AV 的 SL 0–7 直接映射为以太网优先级 0–7〔Vol1 §A17.4.2, p.1938〕。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| PD (Protection Domain) | 保护域 | 〔IB〕关联 QP、AH、MR、MW 的保护机制，只有同一 PD 内的对象可以互相配合使用。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| perftest | perftest 测试工具集 | 〔业界: linux-rdma/perftest〕ib_write_bw、ib_read_lat、ib_send_bw 等 RDMA 带宽/时延测试工具。 | [工具与测试](../07-engineering-practice/04-tools-and-testing.md) |
| PFC (Priority-based Flow Control) | 基于优先级的流控 | 〔外部: IEEE 802.1Qbb〕按 8 个优先级分别暂停的链路级流控（MAC 控制帧 opcode 0x0101），是 RoCEv2 无损网络的主要手段。 | [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md) |
| PFC Deadlock / Storm | PFC 死锁 / 风暴 | 〔业界〕环形缓冲依赖导致的永久暂停，或故障端点持续发送暂停帧造成的大范围阻塞。 | [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md) |
| Pinning Memory | 内存钉住 | 〔IB〕由 OS 保证内存常驻且 VA→PA 映射不变，供 HCA 访问；是传统内存注册的前提（ODP 除外）。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| P_Key (Partition Key) | 分区键 | 〔IB〕16 bit 分区成员标识，最高位 1 为完全成员、0 为受限成员；0xFFFF 为默认分区键（C10-115）。RoCEv2 入站报文仍按第 9 章校验 P_Key〔Vol1 §A17.4.7, p.1940〕。 | [寻址](../00-overview/04-addressing.md) |
| P_Key Table | 分区键表 | 〔IB〕每端口的 P_Key 列表；RoCEv2 端口的填充方法不在规范范围，但至少包含默认 P_Key〔Vol1 §A17.7.1, p.1944〕。 | [寻址](../00-overview/04-addressing.md) |
| PM (Performance Management) | 性能管理 | 〔IB〕MgmtClass 0x04，提供端口计数器等；对 RoCEv2 端口不适用（§A17.6），由以太网/厂商计数器替代。 | [排障指南](../07-engineering-practice/05-troubleshooting.md) |
| PMTU (Path MTU) | 路径 MTU | 〔IB〕源到目的路径支持的最大负载，取值 256/512/1024/2048/4096；RC/UC/XRC 由 QP 属性指定。 | [帧布局](../01-packet-format/01-frame-layout.md) |
| Port | 端口 | 〔IB〕CA 或交换机上连接链路的位置；RoCEv2 端口在 Query HCA 中新增端口类型"RoCEv2"〔Vol1 §A17.5.1, p.1941〕。 | [Verbs 的 RoCEv2 修改](../03-verbs-and-resources/05-verbs-roce-specifics-and-programming.md) |
| Post | 投递 | 〔IB〕把 WR 放入工作队列（Post Send / Post Receive）。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| Private Data | 私有数据 | 〔IB CM〕CM 消息中对 CM 协议不透明的字段，供上层携带附加信息（如 A11 的 IP 地址信息）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| PSN (Packet Sequence Number) | 包序列号 | 〔IB〕BTH 中 24 bit 序号，模 2^24 运算；一个 SQ 未确认的 PSN 不得超过 2^23 个（C9-61）。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |

## 18. Q

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| QCN (Quantized Congestion Notification) | 量化拥塞通知 | 〔外部: IEEE 802.1Qau〕二层拥塞通知标准，DCQCN 的命名与算法来源之一。 | [DCQCN](../05-congestion-control-and-qos/04-dcqcn.md) |
| Q_Key (Queue Key) | 队列密钥 | 〔IB〕UD/RD 中验证发送方访问接收队列权限的 32 bit 值；WR 中 Q_Key 最高位为 1 时改用 QP 上下文中的 Q_Key（C10-15）；QP1 的知名 Q_Key 为 0x80010000（C9-49）。 | [不可靠服务](../02-transport/04-unreliable-services.md) |
| QoS (Quality of Service) | 服务质量 | 〔通用〕IB 中由 SL/VL 实现（Annex A13 对 RoCEv2 不适用）；RoCEv2 中由 DSCP/PCP + ETS 实现〔Vol1 §A17.9.2, p.1946〕。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| QP (Queue Pair) | 队列对 | 〔IB〕一个发送队列加一个接收队列，始终成对创建，由 QPN 标识，是 IB 传输的通信端点。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| QP0 / QP1 | 特殊 QP | 〔IB/A17〕QP0 为子网管理接口（SMI），RoCEv2 端口没有 QP0，DestQP = 0 的报文静默丢弃（CA17-33）；QP1 为 GSI，供 CM 等 GMP 使用。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| QP Context (QPC) | QP 上下文 | 〔IB〕QP 的状态信息（当前 WQE、PSN、传输参数等）；硬件中通常为片上缓存加主机内存后备。 | [RNIC 架构](../06-hardware-implementation/01-rnic-architecture.md) |
| QPN (Queue Pair Number) | QP 号 | 〔IB〕24 bit，标识 CA 内的 QP；0、1 为特殊 QP，0xFFFFFF 用于多播。 | [寻址](../00-overview/04-addressing.md) |
| QP State | QP 状态 | 〔IB〕Reset、Init、RTR、RTS、SQD、SQE、Error 等状态及其迁移规则。 | [QP 状态机](../03-verbs-and-resources/02-qp-state-machine.md) |

## 19. R

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| Raw Datagram | 原始数据报 | 〔IB〕含 LRH、不含 IB 传输头的报文（Raw IPv6 / Raw Ethertype）。RoCEv2 端口不得支持（CA17-28）；注意它与 libibverbs 的 IBV_QPT_RAW_PACKET（收发原始以太网帧的 QP，业界）是不同概念。 | [不可靠服务](../02-transport/04-unreliable-services.md) |
| RC (Reliable Connection) | 可靠连接 | 〔IB〕QP 与唯一对端 QP 相连，消息可靠有序投递，支持 SEND、WRITE、READ、Atomic。 | [服务类型](../02-transport/01-service-types.md) |
| RCM (RoCEv2 Congestion Management) | RoCEv2 拥塞管理 | 〔A17〕可选规范功能：网络以 ECN = CE 标记拥塞，接收端发 CNP，源端按 QP 降速并逐步恢复（CA17-44～CA17-46）。 | [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md) |
| RD (Reliable Datagram) | 可靠数据报 | 〔IB〕QP 经 EE Context 与多个 QP 可靠通信；实际 RoCE 产品几乎不实现（业界）。 | [服务类型](../02-transport/01-service-types.md) |
| RDETH (Reliable Datagram ETH) | 可靠数据报扩展传输头 | 〔IB〕4 字节，携带 24 bit EE Context 号〔Vol1 §9.3.1, p.262〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| RDMA (Remote Direct Memory Access) | 远程直接内存访问 | 〔IB〕无需远端 CPU 参与即可访问远端内存的方法。 | [RDMA 基础](../00-overview/01-rdma-fundamentals.md) |
| rdma_cm (librdmacm) | RDMA 连接管理库 | 〔业界: Linux rdma-core〕以 IP 地址/端口建立 RDMA 连接的 API，底层使用 IB CM 与 A11 私有数据格式。 | [IP CM 与 rdma_cm](../04-connection-management/02-ip-cm-and-rdma-cm.md) |
| rdma-core | rdma-core 软件包 | 〔业界〕Linux 用户态 RDMA 软件包，含 libibverbs、librdmacm、各厂商 provider 与工具。 | [Linux RDMA 软件栈](../07-engineering-practice/01-linux-rdma-stack.md) |
| RDMA READ / RDMA WRITE | RDMA 读 / 写 | 〔IB〕单边内存语义操作：WRITE 把本地数据写入远端，READ 从远端读回；RETH 指定 VA、R_Key 与长度，单次 0～2^31 字节。 | [操作类型](../02-transport/02-operations.md) |
| RED / WRED | 随机早期检测 / 加权随机早期检测 | 〔外部〕按队列长度以线性概率丢包或做 ECN 标记的主动队列管理；交换机 ECN 标记常用其 Kmin/Kmax/Pmax 曲线。 | [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md) |
| REJ (Reject) | 拒绝 | 〔IB CM〕拒绝连接请求或应答；RoCEv2 端口间不得发送原因码 13、19（CA17-35）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| REP (Reply to Request) | 连接应答 | 〔IB CM〕被动方接受 REQ 时的应答，携带本端 QPN、起始 PSN 等（AttributeID 0x0013）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| REQ (Request for Communication) | 连接请求 | 〔IB CM〕主动方发起的连接请求（AttributeID 0x0010）；RoCEv2 中 4 个 LID 字段保留且接收时忽略（CA17-34）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| Requester / Responder | 请求方 / 响应方 | 〔IB〕发起 SEND/WRITE/READ/Atomic 请求的一端 / 执行请求并返回响应的一端。 | [请求方](../02-transport/06-requester.md) |
| RespTimeValue | 响应时间值 | 〔IB〕MAD 响应时间上界编码，时间 = 4.096 µs × 2^值〔Vol1 §13.4.6.2.2, p.791〕；RoCE v1 默认 19（CA16-9）。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| RESYNC | 重同步 | 〔IB〕RD 专用的零长度请求，用于重同步 EE Context 的 PSN。 | [操作类型](../02-transport/02-operations.md) |
| RETH (RDMA ETH) | RDMA 扩展传输头 | 〔IB〕16 字节：VA(64) + R_Key(32) + DMALen(32)，出现在 WRITE First/Only 与 READ Request 中〔Vol1 §9.3.3, p.263〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| Retry Count | 重试计数 | 〔IB〕3 bit，请求方因超时或 PSN 序列错 NAK 重传的最大次数，耗尽后报告错误（C9-142）。 | [请求方](../02-transport/06-requester.md) |
| R_Key (Remote Key) | 远端密钥 | 〔IB〕注册 MR 或绑定 MW 时生成，远端在 RETH/AtomicETH 中携带以获得访问授权。 | [内存管理](../03-verbs-and-resources/03-memory-management.md) |
| RMPP (Reliable Multi-Packet Protocol) | 可靠多包协议 | 〔IB〕基于 MAD、传输超过单个 MAD 容量数据的事务协议。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| RNIC (RDMA NIC) | RDMA 网卡 | 〔业界〕支持 RDMA 的以太网卡，在 IB 规范语境下即 RoCE/RoCEv2 HCA。 | [RNIC 架构](../06-hardware-implementation/01-rnic-architecture.md) |
| RNR NAK (Receiver Not Ready) | 接收方未就绪 NAK | 〔IB〕响应方暂无可用接收 WQE 时返回（Syndrome[6:5] = 01），低 5 bit 编码请求方须等待的最短时间（Table 48）。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |
| RNR Retry Count | RNR 重试计数 | 〔IB〕3 bit，收到 RNR NAK 后的重试次数，7 表示无限重试（C9-132）。 | [请求方](../02-transport/06-requester.md) |
| RoCE (RoCE v1) | 融合以太网上的 RDMA（第一版） | 〔A16〕以太网 MAC 头 + IB GRH + IB 传输层，Ethertype 0x8915，只能在二层域内通信〔Vol1 §A17.2.3, p.1929〕。 | [RoCE 演进](../00-overview/02-roce-evolution.md) |
| RoCEv2 (IP Routable RoCE) | 可路由 RoCE | 〔A17〕以 IP 头替代 GRH 并加 UDP 头（目的端口 4791）的 RoCE，可经 IP 路由器转发〔Vol1 §A17.2.4, p.1930〕。 | [RoCE 演进](../00-overview/02-roce-evolution.md) |
| RP (Reaction Point) | 反应点 | 〔论文: DCQCN〕DCQCN 中的发送端角色：收到 CNP 后降速，再按定时器/字节计数器恢复。 | [DCQCN](../05-congestion-control-and-qos/04-dcqcn.md) |
| RQ (Receive Queue) | 接收队列 | 〔IB〕QP 中存放接收 WQE 的队列，描述入站数据的放置位置。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| RS-FEC (Reed-Solomon FEC) | 里德-所罗门前向纠错 | 〔IB Vol2/外部: IEEE 802.3〕多 lane 联合编码的纠错码：RS(528,514) 用于 IB EDR 与 100GBASE-R，RS(544,514) 用于 IB HDR 与 50G/lane PAM4 以太网。 | [IB 物理层对比](04-ib-physical-layer-vs-ethernet.md) |
| RTU (Ready To Use) | 就绪可用 | 〔IB CM〕主动方收到 REP 后发出，通知被动方连接可用（AttributeID 0x0014）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |

## 20. S

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| SA (Subnet Administration) | 子网管理服务 | 〔IB〕查询/操作子网管理数据（如 PathRecord）的接口；RoCEv2 不适用（§A17.6）。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| SAP / SPR | 建议备用路径 / 建议路径响应 | 〔IB CM〕Suggest Alternate Path / Suggest Path Response；RoCEv2 中 SAP 的 Alternate Local Port LID 保留并在接收时忽略（CA17-38）。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| SAR (Segmentation and Reassembly) | 分段与重组 | 〔IB〕按 PMTU 把消息拆成 First/Middle/Last 包，接收端按序重组。 | [操作类型](../02-transport/02-operations.md) |
| SE (Solicited Event) | 请求事件位 | 〔IB〕BTH 中 1 bit，只应在 SEND（含带立即数/带失效）或 RDMA WRITE with Immediate 的最后或唯一包置 1，请求接收端触发 CQ 事件处理函数〔Vol1 §9.2.3, p.260〕。 | [BTH](../01-packet-format/03-bth.md) |
| Service ID | 服务 ID | 〔IB〕64 bit，CM 用它把连接请求关联到具体服务（类似 TCP 端口号）；A11 定义 IP 端口到 Service ID 的映射。 | [IP CM 与 rdma_cm](../04-connection-management/02-ip-cm-and-rdma-cm.md) |
| SGE (Scatter/Gather Element) | 分散/聚合元素 | 〔业界/IB〕Verbs 中描述一段本地缓冲（addr, length, lkey）的结构，即规范中的 Data Segment。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| SGID (Source GID) | 源 GID | 〔IB/A17〕源端口 GID；RoCEv2 中不直接存于 AV，而经 SGID index 引用端口 GID 表〔Vol1 §A17.4.2, p.1937〕。 | [寻址](../00-overview/04-addressing.md) |
| SIDR (Service ID Resolution) | 服务 ID 解析 | 〔IB CM〕SIDR_REQ / SIDR_REP，用于 UD 场景查询服务对应的 QPN 与 Q_Key。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| Signaled / Unsignaled Completion | 有信号 / 无信号完成 | 〔IB〕发送 WR 是否总是产生 CQE；无信号 WR 只在出错时产生完成。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| Silent Drop | 静默丢弃 | 〔IB〕丢弃报文且不回 NAK、不产生 CQE（可计数）；RoCEv2 中格式不合规、ICRC 错、IP 版本错、DestQP = 0 等均如此处理。 | [排序与校验](../02-transport/03-ordering-and-validation.md) |
| SL (Service Level) | 服务等级 | 〔IB/A17〕IB 中为 LRH 的 4 bit 字段；RoCEv2 中 AV 的 SL 0–7 映射为以太网优先级 0–7，SL 8–15 保留〔Vol1 §A17.4.2, p.1938〕。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| SLID (Source LID) | 源 LID | 〔IB〕LRH 中的源 LID；RoCEv2 不适用。 | [寻址](../00-overview/04-addressing.md) |
| SM (Subnet Manager) | 子网管理器 | 〔IB〕配置 IB 子网（LID 分配、转发表等）的实体；RoCEv2 没有 SM，使用以太网/IP 管理〔Vol1 §A17.6, p.1942–1943〕。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| SMA (Subnet Management Agent) | 子网管理代理 | 〔IB〕每个 CA/交换机中经 QP0 处理 SMP 的实体；RoCEv2 不需要。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| SMP (Subnet Management Packet) | 子网管理包 | 〔IB〕只走 VL15、只发往 QP0 的管理 MAD；RoCEv2 不适用。 | [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md) |
| Soft-RoCE (rxe) | 软件 RoCE | 〔业界: Linux 内核 rdma_rxe〕在普通以太网卡上用软件实现 RoCEv2 的驱动，适合学习、调试和互通测试。 | [Linux RDMA 软件栈](../07-engineering-practice/01-linux-rdma-stack.md) |
| SQ (Send Queue) | 发送队列 | 〔IB〕QP 中存放发送 WQE 的队列。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| SR-IOV (Single Root I/O Virtualization) | 单根 I/O 虚拟化 | 〔外部: PCI-SIG〕把一个 PCIe 设备虚拟成多个 VF，供虚拟机直通使用 RDMA（业界常见用法）。 | [典型场景](../07-engineering-practice/06-use-cases.md) |
| SRQ (Shared Receive Queue) | 共享接收队列 | 〔IB〕可被多个 QP 共享的接收 WQE 池；关联 SRQ 的 HCA 接收队列不生成端到端信用（C17-15.2.1）。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| SSN (Send Sequence Number) | 发送序列号 | 〔IB〕请求方为每个发送 WQE 逻辑分配的序号，与响应方返回的 MSN 一一对应〔Vol1 §9.7.7.2, p.379〕。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |
| Static Rate Control | 静态速率控制 | 〔IB〕按 IPD 限制注入速率，使快端口不压垮慢路径〔Vol1 §9.11, p.456〕。 | [请求方](../02-transport/06-requester.md) |
| Swift | Swift | 〔论文: Swift, SIGCOMM 2020〕以时延为信号的数据中心拥塞控制。 | [高级拥塞控制](../05-congestion-control-and-qos/05-advanced-cc-and-lossy-roce.md) |
| Switch | 交换机 | 〔IB〕按 LRH 的 DLID 在子网内转发的设备；RoCEv2 使用普通以太网交换机与 IP 路由器。 | [分层架构](../00-overview/03-architecture-and-layering.md) |
| Syndrome (AETH) | Syndrome 字段 | 〔IB〕AETH 高 8 bit：[6:5] = 00 ACK（低 5 bit 为信用）、01 RNR NAK（定时器）、10 保留、11 NAK（NAK 码），bit7 保留为 0〔Vol1 Table 46, p.348〕。 | [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) |

## 21. T

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| TCA (Target Channel Adapter) | 目标通道适配器 | 〔IB〕通常用于 I/O 设备的 CA，不要求支持 Verbs 接口。 | [分层架构](../00-overview/03-architecture-and-layering.md) |
| TIMELY | TIMELY | 〔论文: TIMELY, SIGCOMM 2015〕以 RTT 梯度为信号的 RDMA 拥塞控制。 | [高级拥塞控制](../05-congestion-control-and-qos/05-advanced-cc-and-lossy-roce.md) |
| TimeWait | 时间等待 | 〔IB CM〕连接释放后 QP 须保持的静默期（2 × PacketLifeTime + 对端 ACK 延迟），防止旧报文混入新连接〔Vol1 §12.9.8.4, p.757〕。 | [CM 协议](../04-connection-management/01-cm-protocol.md) |
| Traffic Class (TClass) | 流量类别 | 〔IB/A17〕AV/GRH 中 8 bit 字段；RoCEv2 中其值写入 IP 的 DSCP（CA17-4、CA17-13），IPv6 Traffic Class 在 ICRC 中被屏蔽。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| Transport Service Type | 传输服务类型 | 〔IB〕描述可靠性、顺序、消息大小与可用操作的服务类别：RC、UC、RD、UD（及 XRC）；Raw 不使用 IBA 传输。 | [服务类型](../02-transport/01-service-types.md) |
| Transport Timer (Ttr) | 传输定时器 | 〔IB〕RC 请求方等待确认的定时器：Ttr = 4.096 µs × 2^(Local ACK Timeout)，超时须在 Ttr～4Ttr 之间检测到（C9-140、C9-141）。 | [请求方](../02-transport/06-requester.md) |
| Trust Mode (PCP / DSCP Trust) | 信任模式 | 〔业界〕交换机或网卡依据 PCP 还是 DSCP 判定报文优先级的配置；RoCEv2 跨三层时通常使用 DSCP trust（厂商相关）。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| TTL (Time to Live) | 生存时间 | 〔外部: RFC 791〕IPv4 跳数字段；RoCEv2 中取自 AV 的 Hop Limit（CA17-9），ICRC 中屏蔽。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |
| TVer (Transport Header Version) | 传输头版本 | 〔IB〕BTH 中 4 bit，当前必须为 0（C9-5）；接收端不支持的版本被丢弃。 | [BTH](../01-packet-format/03-bth.md) |

## 22. U

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| UC (Unreliable Connection) | 不可靠连接 | 〔IB〕一对一连接，支持 SEND 与 RDMA WRITE，出错不重传，由上层处理。 | [不可靠服务](../02-transport/04-unreliable-services.md) |
| UD (Unreliable Datagram) | 不可靠数据报 | 〔IB〕单包消息、可与任意 QP 通信，不保证投递与顺序；RoCEv2 UD 接收缓冲前 40 字节存放 IP 头〔Vol1 §A17.4.5.2, p.1939〕。 | [不可靠服务](../02-transport/04-unreliable-services.md) |
| UDP Source Port (sport) | UDP 源端口 | 〔A17〕RoCEv2 中作为不透明流标识供 ECMP 哈希；有顺序要求的报文（如同一连接 QP）应使用固定值〔Vol1 §A17.3.2.1, p.1934〕。 | [ECMP](../05-congestion-control-and-qos/06-ecmp-and-load-balancing.md) |
| UEC (Ultra Ethernet Consortium) | 超以太网联盟 | 〔外部〕制定面向 AI/HPC 的新一代以太网传输规范的产业组织，与 RoCEv2 是不同规范。 | [高级拥塞控制](../05-congestion-control-and-qos/05-advanced-cc-and-lossy-roce.md) |
| ULP (Upper Layer Protocol) | 上层协议 | 〔通用〕使用 Verbs 的上层协议，如 NVMe-oF、iSER、SMB Direct、NCCL。 | [典型场景](../07-engineering-practice/06-use-cases.md) |

## 23. V

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| VCRC (Variant CRC) | 可变 CRC | 〔IB〕覆盖整个 IB 包的 16 bit CRC（多项式 0x100B），逐跳重算〔Vol1 §7.8.2, p.219–220〕；RoCE/RoCEv2 不含 VCRC，由以太网 FCS 代替。 | [ICRC](../01-packet-format/06-icrc.md) |
| Verbs | Verbs 接口 | 〔IB〕对 HCA 功能的抽象描述；OS 通过编程接口（如 libibverbs）暴露其中部分或全部功能。 | [Verbs 的 RoCEv2 修改](../03-verbs-and-resources/05-verbs-roce-specifics-and-programming.md) |
| Verbs Consumer | Verbs 使用者 | 〔IB〕直接调用 Verbs 的软件（应用或 ULP）。 | [Verbs 的 RoCEv2 修改](../03-verbs-and-resources/05-verbs-roce-specifics-and-programming.md) |
| VL (Virtual Lane) | 虚通道 | 〔IB〕同一物理链路上的独立逻辑流（VL0–14 为数据、VL15 为子网管理），各自做信用流控〔Vol1 §7.6, p.199–201〕；RoCEv2 中由以太网优先级/队列取代。 | [QoS](../05-congestion-control-and-qos/02-qos-dscp-pcp-ets.md) |
| VLAN | 虚拟局域网 | 〔外部: IEEE 802.1Q〕4 字节标签（TPID 0x8100 + PCP/DEI/VID）；RoCEv2 二层优先级（PCP）依赖 VLAN 标签。 | [以太网/IP/UDP](../01-packet-format/02-ethernet-ip-udp.md) |

## 24. W

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| WC (Work Completion) | 工作完成 | 〔IB〕消费者从 CQ 中取得的、对 CQE 的可见表示。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| WQ (Work Queue) | 工作队列 | 〔IB〕发送队列或接收队列的统称。 | [资源模型](../03-verbs-and-resources/01-resource-model.md) |
| WQE (Work Queue Element) | 工作队列元素 | 〔IB〕WR 在 HCA 内部的表示，俗称"wookie"，消费者不能直接访问。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |
| WR (Work Request) | 工作请求 | 〔IB〕消费者请求创建 WQE 的方式（Post Send / Post Receive）。 | [WR 与完成](../03-verbs-and-resources/04-work-requests-and-completions.md) |

## 25. X

| 术语 | 中文 | 解释 | 详解 |
|---|---|---|---|
| Xoff / Xon | 暂停 / 恢复阈值 | 〔业界〕交换机入口缓冲触发发出 / 撤销 PFC 暂停的水位（厂商相关）。 | [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md) |
| XRC (eXtended Reliable Connected) | 扩展可靠连接 | 〔IB〕用 XRC SRQ 共享接收资源、减少全互联所需 QP 数的可靠传输服务（OpCode 高 3 bit 101）。 | [服务类型](../02-transport/01-service-types.md) |
| XRC Domain | XRC 域 | 〔IB〕关联 XRC TGT QP 与 XRC SRQ 的属性；XRC 报文只能命中与目标 TGT QP 同域的 XRC SRQ。 | [服务类型](../02-transport/01-service-types.md) |
| XRCETH | XRC 扩展传输头 | 〔IB〕4 字节：保留(8) + XRCSRQ(24)，出现在 XRC 请求报文中〔Vol1 §9.3.8, p.267〕。 | [扩展传输头](../01-packet-format/04-extended-transport-headers.md) |
| XRC INI QP / XRC TGT QP | XRC 发起 QP / XRC 目标 QP | 〔IB〕只有请求方能力的发起 QP / 只有响应方能力的目标 QP。 | [服务类型](../02-transport/01-service-types.md) |
| XRC SRQ | XRC 共享接收队列 | 〔IB〕XRC 请求报文通过 XRCETH 指定的接收 WQE 来源。 | [服务类型](../02-transport/01-service-types.md) |

## 26. 要点回顾

- RoCEv2 沿用 IB 传输层与 Verbs 的全部术语（QP、PSN、BTH、AETH、R_Key……），但**链路层与网络层术语**（LRH、LID、VL、GRH、SM、SMP、QP0）在 RoCEv2 中不适用或被以太网/IP 概念替代。
- 规范把 RoCEv2 的 IP 地址统称为 **GID**（IPv4 用 ::ffff:x.x.x.x 映射），并为 GID 表项新增 **GID type** 属性。
- 拥塞相关术语要区分三套体系：IB CC（FECN/BECN/CCT，Annex A10）、RoCEv2 RCM（ECN/CNP，Annex A17）、具体算法（DCQCN 等，论文/厂商）。
- 以太网侧的关键术语：PFC、ETS、DCBX、DSCP、PCP、ECN、ECMP、Headroom；它们大多不是 IBTA 规范内容，需要参照 IEEE/IETF 标准与厂商文档。
- "ETH"在 IB 语境中是 Extended Transport Header，不要与 Ethernet 混淆。

## 参考

- 规范：Vol1 第 2 章 Glossary（p.84–100）；Vol1 Annex A17（p.1928–1949）；Vol1 Annex A16（p.1911–1927）；Vol1 Annex A10 §A10.1.1（p.1797–1799）；Vol2 第 2 章 Glossary（p.40–61）。
- 外部：IEEE 802.1Q / 802.1Qbb / 802.1Qaz / 802.1Qau / 802.1AB / 802.3；RFC 791、2474、3168、4861、8200、5040/5041/5044、7145。
- 论文：DCQCN（SIGCOMM 2015）、TIMELY（SIGCOMM 2015）、IRN（SIGCOMM 2018）、HPCC（SIGCOMM 2019）、Swift（SIGCOMM 2020）。
- 业界实践：Linux rdma-core / 内核 RDMA 子系统（rxe、ib_addr.h、ib_verbs.h）；linux-rdma/perftest。
