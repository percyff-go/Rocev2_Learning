# FAQ 与自测题（FAQ and Self-Test）

> **摘要**：58 个按主题分组的常见问题与自测题，覆盖概念与演进、报文格式、ICRC、传输与可靠性、Verbs 与 CM、拥塞控制与 QoS、硬件实现、运维排障，特别收录了一批容易混淆的问题（UDP 校验和为何为 0、ICRC 为何屏蔽 TTL、v1/v2 GID 区别、CNP 的 DestQP 填谁、为何固定 UDP 源端口、FECN 是否还用……）。每题先自己作答，再展开参考答案；答案附规范出处或知识库链接。
>
> **规范出处**：Vol1 Annex A17（p.1928–1949）、Annex A16（p.1911–1927）、第 9 章（p.253–459）、§7.8（p.218–231）、Annex A10（p.1797–1847）、Annex A11（p.1848–1856）
>
> **前置阅读**：建议先通读 [RoCE 演进](../00-overview/02-roce-evolution.md)、[帧布局](../01-packet-format/01-frame-layout.md)、[合规清单](03-compliance-checklist.md)
>
> **关键词**：FAQ, 自测, RoCEv2, ICRC, CNP, ECN, PFC, PSN, GID, UDP source port

## 使用方法

- 每题先遮住答案自己回答，再点开 `<details>` 对照；答案中的〔Vol1 §x, p.N〕可直接在 PDF 中定位。
- 标"（解读）"的内容是根据规范推理得出的工程判断，标〔业界〕的内容与具体实现或厂商相关。
- 涉及计算的题目，常数与公式见 [常数与公式](05-constants-and-formulas.md)。

## 1. 概念与演进

<details><summary>Q1：RoCEv2 与 RoCE（v1）的本质区别是什么？</summary>

答：RoCE v1 是"以太网 MAC 头（Ethertype 0x8915）+ IB GRH + IB 传输层"，没有 IP 头，因而不能被普通 IP 路由器跨二层子网转发，只能在一个以太网二层域内通信〔Vol1 §A17.2.3, p.1929〕。RoCEv2 用 **IP 头替代 GRH**，并加一个 **UDP 头**作为无状态封装层，用知名 UDP 目的端口识别 RoCEv2，因而可以跨三层路由〔Vol1 §A17.2.4, p.1930〕。BTH 及以后的传输层、Verbs 语义完全相同。详见 [RoCE 演进](../00-overview/02-roce-evolution.md)。
</details>

<details><summary>Q2：RoCEv2 是 IB 规范的强制内容吗？"Optional Normative"是什么意思？</summary>

答：不是强制内容。Annex A17 是"可选规范"：是否实现由产品决定，但一旦实现就必须满足附录中的合规条款（CA17-x）〔Vol1 §A17.1, p.1928〕。附录内部还有两层可选：RCM（拥塞管理）是可选规范行为〔Vol1 §A17.9.3, p.1947〕；与 RoCE v1 互通是可选能力〔Vol1 §A17.8, p.1945〕。
</details>

<details><summary>Q3：RoCEv2 保留了 IB 的哪些部分，替换了哪些？</summary>

答：保留：传输层（第 9 章，CA17-25）、Verbs（按 §A17.5 修改，CA17-30）、通信管理 CM（第 12 章，去掉 LID 相关内容，§A17.6.1）、内存与保护模型。替换：物理层与链路层换成以太网〔Vol1 §A17.2.2, p.1928〕；网络层 GRH 换成 IP/UDP〔Vol1 §A17.2.4〕；SM/SA/PM 等 IB 管理类换成以太网/IP 管理实践〔Vol1 §A17.6, p.1942–1943〕；IB 拥塞控制（A10）与 IB QoS（A13）不适用，改用 ECN/CNP 与 DSCP/PCP/ETS〔Vol1 §A17.4.8–A17.4.9、§A17.9〕。
</details>

<details><summary>Q4：RoCEv2 和 iWARP 有什么区别？</summary>

答：iWARP 是 IETF 定义的 RDMAP/DDP/MPA 协议栈，跑在 TCP 之上〔外部: RFC 5040/5041/5044〕，可靠性、拥塞控制都依赖 TCP，天然适应有损网络，但协议处理更重。RoCEv2 跑在 UDP 之上，可靠性由 IB 传输层（PSN、ACK/NAK、Go-back-N 重传）提供，规范建议网络配置为无损并可选 ECN/CNP 拥塞管理〔Vol1 §A17.9〕。两者在 Linux 上都通过 libibverbs/rdma_cm 暴露相似的编程接口〔业界〕。详见 [RDMA 基础](../00-overview/01-rdma-fundamentals.md)。
</details>

<details><summary>Q5：RoCE v1 与 RoCEv2 的 GID 有什么区别？</summary>

答：两者都是端口 GID 表中的 128 bit GID。RoCE v1 的 GID 放在 GRH 的 SGID/DGID 中，GID 到 MAC 的解析不在规范范围〔Vol1 §A16.5.1, p.1916〕；Linux 中常见的 v1 GID 包括由 MAC 派生的链路本地地址 fe80::/64 以及基于 IP 的 GID〔业界〕。RoCEv2 的 GID **就是 IP 地址**：IPv6 原样作为 GID，IPv4 编码为 ::ffff:a.b.c.d〔Vol1 §A17.4.1.1, p.1936〕，并且表项带"GID type"属性（IPv4/IPv6/IB GID，CA17-26）——决定封装协议的是**类型**而不是 GID 数值〔Vol1 §A17.8〕。Linux 会为同一个 IP 同时生成"IB/RoCE v1"和"RoCE v2"两个类型不同的表项，选错 GID index 就会用错协议〔业界: sysfs gid_attrs/types〕。详见 [寻址](../00-overview/04-addressing.md)。
</details>

<details><summary>Q6：一块 RoCEv2 网卡能和 RoCE v1 节点通信吗？</summary>

答：可以，但这是可选能力。支持时端口的"RoCE Supported"能力位置位〔Vol1 §A17.5.1, p.1941〕；连接型 QP 按所配 GID 表项的类型工作在 v1 或 v2 模式（CA17-40），UD QP 按 AV 引用的表项类型逐包选择（CA17-42），并同时接受两种报文（CA17-43）；DGID 检查时类型与报文协议不符视为失败（CA17-41）。与 v1 端点建连时，CM 报文本身也是 RoCE v1 报文〔Vol1 §A17.6.1, p.1943〕。
</details>

<details><summary>Q7：为什么 RoCEv2 选择 UDP 封装，而不是申请一个新的 IP 协议号？</summary>

答：规范给出的理由：UDP 头是无状态的封装层，知名目的端口能无状态地识别 RoCEv2；按照"UDP 封装协议"的惯例，源端口可作为不透明的流标识，供网络做转发优化（ECMP）〔Vol1 §A17.2.4, p.1930〕。工程上（解读）：交换机/路由器普遍支持对 UDP 五元组做 ECMP 哈希，ACL、监控也都识别 UDP，新协议号很难获得同等支持。
</details>

## 2. 报文格式

<details><summary>Q8：RoCEv2 的 UDP 目的端口是多少？为什么规范正文没有写数值？</summary>

答：4791。CA17-20 只要求"设置为 IANA 为 RoCEv2 分配的值"，脚注 60 说明"分配后再更新"〔Vol1 §A17.3.2.2, p.1934–1935〕；IANA 后来分配了 4791〔外部: IANA〕。接收端用它把 RoCEv2 报文与普通 UDP 流量区分开。
</details>

<details><summary>Q9：RoCEv2 的 UDP checksum 为什么是 0？接收端要不要校验？</summary>

答：规范只说 UDP 校验和"should 置 0"〔Vol1 §A17.3.2.4, p.1935〕。原因（解读）：① 端到端完整性已经由覆盖 UDP 头与负载的 ICRC（CRC-32）保证，比 16 bit 反码和更强；② UDP 校验和位于头部却要覆盖整个负载，发送端必须先扫完全部数据才能填头，不利于直通（cut-through）硬件；③ IPv4 中 0 表示"未计算校验和"〔外部: RFC 768〕。ICRC 计算时 UDP 校验和被替换为全 1（CA17-22(d)），所以即使中间设备填写了它也不影响 ICRC。接收端不需要校验，也不宜因其非 0 而丢包（解读）。

> [!WARNING]
> IPv6 下 RFC 8200 原则上要求 UDP 校验和非 0，零校验和只在 RFC 6935/6936 规定的隧道场景下放宽〔外部〕；个别主机协议栈或中间设备可能丢弃零校验和的 IPv6 UDP 报文，跨域部署时需要留意。
</details>

<details><summary>Q10：为什么 IPv4 头必须 IHL = 5、DF = 1、Fragment Offset = 0？</summary>

答：这是 CA17-3、CA17-7、CA17-8 的要求。理由（解读）：固定 20 字节 IPv4 头便于硬件定长解析；IB 传输层已经按 PMTU 分段，每个 IB 包都要独立校验 ICRC、独立按 PSN 处理，IP 分片后的后续分片没有 UDP/BTH，无法处理；置 DF 让路径 MTU 不足时报文被丢弃而不是被分片，问题会以"丢包/重传"暴露出来，所以部署时必须保证全路径 MTU 足够。
</details>

<details><summary>Q11：IP 头中的 DSCP、TTL 和以太网优先级（PCP）分别从哪里来？</summary>

答：都来自 Address Vector：DSCP = AV 的 Traffic Class（CA17-4、CA17-13）；IPv4 TTL / IPv6 Hop Limit = AV 的 Hop Limit（CA17-9、CA17-17）；以太网优先级 = AV 的 SL（SL 0–7 直接映射到优先级 0–7，SL 8–15 保留）〔Vol1 §A17.4.2, p.1938〕。源 IP 来自 SGID index 指向的 GID 表项，目的 IP 来自 DGID（CA17-11/12/18/19）；源 MAC 由所选 SGID 隐含，目的 MAC 由 L3→L2 解析或 AV 的"目的 L2 地址"给出〔Vol1 §A17.4.2, p.1937〕。
</details>

<details><summary>Q12：RoCEv2 报文里还有 LRH、VL、DLID/SLID 吗？</summary>

答：没有。规范中所有对 LRH 及其字段的引用对 RoCEv2 端口都不适用〔Vol1 §A17.4.1.2, p.1936〕。AV 的 DLID 被泛化为"目的 L2 地址"（DMAC），Source Path Bits 不适用，SL 映射为以太网优先级〔Vol1 §A17.4.2〕；VL 的角色由以太网优先级与交换机队列承担。
</details>

<details><summary>Q13：IPv6 RoCEv2 报文的 Payload Length 应该怎么算？</summary>

答：**Payload Length = UDP Length** = UDP 头 8 + BTH 12 + 扩展头 + 负载 + pad + ICRC 4。CA17-15 原文写"从 IPv6 头之后第一个字节（即 BTH）算到 ICRC"〔Vol1 §A17.3.1.2.3, p.1934〕，括号里的"即 BTH"是笔误——IPv6 头之后紧跟的是 UDP 头；按 IPv6 定义（RFC 8200）与 Linux rxe 等实际实现，长度应包含 UDP 头〔业界〕。计算方法见 [常数与公式](05-constants-and-formulas.md)。
</details>

<details><summary>Q14：一个 4 KB 的 RDMA WRITE Only 报文（IPv4、无 VLAN）在线上占多少字节？效率是多少？</summary>

答：帧长 = 14 + 20 + 8 + 12（BTH）+ 16（RETH）+ 4096 + 4（ICRC）+ 4（FCS）= 4174 字节，加前导码/SFD 8 与 IFG 12 后线上占用 4194 字节，负载效率 4096/4194 ≈ 97.66%。不带 RETH 的 Middle 包为 4178 字节、约 98.04%。
</details>

<details><summary>Q15：BTH 的 PadCnt 什么时候不为 0？pad 字节算不算在长度和 ICRC 里？</summary>

答：负载不是 4 字节整数倍时才需要 1–3 个 pad〔Vol1 §9.2.5, p.260〕。由于 First/Middle 包的负载都等于 PMTU（4 的倍数）〔Vol1 §9.4.4, p.281〕，pad 只可能出现在 Last 或 Only 包。pad 属于负载的一部分：计入 UDP Length、IP 长度（它们都算到 ICRC 为止，CA17-6、CA17-21），也被 ICRC 覆盖。
</details>

## 3. ICRC

<details><summary>Q16：有了以太网 FCS，为什么还需要 ICRC？</summary>

答：FCS 是逐跳的：每台交换机/路由器收包校验、发包重算，设备内部（缓冲、转发流水线）发生的比特翻转会被"合法地"重新算进新的 FCS。ICRC 是端到端的，只覆盖途中不变的字段，规范称它"提供超出以太网 FCS（通常逐跳更新）覆盖范围的保护"〔Vol1 §A17.3.3, p.1935〕。加上 RoCEv2 的 UDP 校验和为 0，ICRC 就是唯一的端到端数据完整性校验。
</details>

<details><summary>Q17：为什么 ICRC 要屏蔽 TTL？还屏蔽了哪些字段？</summary>

答：ICRC 只能覆盖端到端不变的字段。TTL / Hop Limit 每经过一个路由器减 1，IPv4 头校验和随之改变；DSCP 可能被 QoS 策略改写，ECN 会被拥塞的交换机标记为 CE；IPv6 Flow Label 也可能被改写。因此计算 ICRC 时把这些字段替换为全 1：IPv4 屏蔽 TTL、Header Checksum、TOS（DSCP+ECN）；IPv6 屏蔽 Traffic Class、Flow Label、Hop Limit；另外屏蔽 UDP Checksum（CA17-22(c)(d)）与 BTH 中 F/B/Resv6 所在的 Resv8a 字节〔Vol1 §7.8.1, p.218–219〕。如果不屏蔽 ECN，RCM 本身（交换机标记 CE）就会让 ICRC 失败。
</details>

<details><summary>Q18：ICRC 计算开头那 64 bit 全 1 是从哪来的？</summary>

答：为了与 RoCE v1 等价：v1 的 ICRC 先把 64 bit 全 1（一个"假 LRH"）送入 CRC 机，再送 GRH〔Vol1 §A17.3.3 脚注 61, p.1935〕；而在带 GRH 的 IB 报文中，LRH 全部字段本来就被替换为 1〔Vol1 §7.8.1, p.218〕。这样 IB、RoCE、RoCEv2 三者的 ICRC 计算起点一致。硬件/软件实现常把"已输入 8 字节 0xFF 后的 CRC 状态"预先算好作为初值〔业界〕。
</details>

<details><summary>Q19：UDP 源端口、IP 地址参与 ICRC 吗？RoCEv2 能穿越 NAT 吗？</summary>

答：参与。CA17-22 的屏蔽列表只有 TTL/TOS/头校验和（IPv4）、Traffic Class/Flow Label/Hop Limit（IPv6）与 UDP 校验和，UDP 端口、IP 地址都被覆盖。因此改写地址或端口的 NAT、某些四层负载均衡器会让 ICRC 校验失败，报文被静默丢弃（CA17-24）；再加上 rdma_cm 在 CM 私有数据里携带 IP 地址（A11），RoCEv2 一般不能穿越 NAT（解读）。
</details>

<details><summary>Q20：ICRC 错误的报文怎么处理？会触发 NAK 吗？</summary>

答：静默丢弃，不回 NAK、不产生 CQE，只增加丢弃计数（CA17-24）。对 RC 而言，丢包由后续机制恢复：后面的请求到达时 PSN 超前，响应方回一次 NAK（PSN Sequence Error）并等待期望的 PSN〔C9-79、C9-80, p.325〕；若后面没有报文，则靠请求方传输定时器超时重传（C9-140）。
</details>

<details><summary>Q21：如何快速验证自己的 ICRC 实现？</summary>

答：分三步：① 用 §7.8.4 的两个样例（Table 28 结果 0x9625B75A、Table 33 结果 0xB67D1ED1，p.226/p.230）验证 CRC 内核与比特序；② 参数与以太网 CRC-32 相同，可用 `zlib.crc32` 作软件参考模型，线上 4 字节等于 CRC 值的小端字节序（[常数与公式](05-constants-and-formulas.md) 中的代码已用 Table 28 验证）；③ 用真实 RoCEv2 抓包或 Linux rxe 互发，验证 64 bit 前缀与屏蔽字段。详见 [ICRC](../01-packet-format/06-icrc.md)。
</details>

## 4. 传输与可靠性

<details><summary>Q22：为什么规范建议 RoCEv2 网络"无损"？没有 PFC 就不能用吗？</summary>

答：IB 传输层虽然有端到端重传，但它是为几乎不丢包的链路设计的，丢包代价高（Go-back-N 会重传丢失点之后的全部报文）。规范建议把网络配置为无损（通常用 802.1Qbb PFC），并强调"无损不等于绝不丢包"，目的是避免因网络争用而丢包〔Vol1 §A17.9.1, p.1946〕。没有 PFC 也能工作，但吞吐对丢包率非常敏感；业界有"有损 RoCE"方案（选择性重传、更强的拥塞控制等）〔业界/论文: IRN〕。详见 [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md)、[有损 RoCE](../05-congestion-control-and-qos/05-advanced-cc-and-lossy-roce.md)。
</details>

<details><summary>Q23：Local ACK Timeout 设为 14 意味着多长时间？设得太小或为 0 有什么问题？</summary>

答：Ttr = 4.096 µs × 2^14 ≈ 67.1 ms，实际超时会在 Ttr～4Ttr（约 67～268 ms）之间被检测到（C9-140、C9-141, p.364）。设得太小，PFC 暂停或拥塞排队就会引发虚假超时与重传；设得太大，真实丢包的恢复变慢〔Vol1 §12.7.34, p.737–738〕。设为 0 表示**禁用**定时器（C9-140），一旦尾包丢失 QP 可能永远等待。
</details>

<details><summary>Q24：RNR NAK 与普通 NAK 有什么区别？RNR 定时器编码 0 表示什么？</summary>

答：RNR NAK（Syndrome[6:5] = 01）表示响应方暂时没有接收 WQE，属于"稍后再试"：请求方至少等待 AETH 中编码的时间再重发同一请求（C9-130），次数由 RNR Retry Count 限制，7 表示无限（C9-132）。普通 NAK（Syndrome[6:5] = 11）带 NAK 码：PSN 序列错可回退重传，Invalid Request、Remote Access Error、Remote Operational Error 属于错误〔Vol1 Table 47, p.349〕。RNR 定时器编码 00000 不是 0，而是**最长的 655.36 ms**〔Vol1 Table 48, p.354〕。
</details>

<details><summary>Q25：为什么 RDMA READ 请求之后，下一个请求的 PSN 要"跳号"？</summary>

答：READ 响应包也消耗 PSN：一个 READ 预期有 n 个响应包，请求方就为它预留 n 个 PSN，下一个请求的 PSN = READ 的 PSN + n（模 2^24），从而保证所有响应包的 PSN 单调递增〔Vol1 Table 42, p.314〕。n = max(1, ⌈DMALen / PMTU⌉)，零长度 READ 也有一个 Response Only〔Vol1 §9.4.4, p.281〕。
</details>

<details><summary>Q26：PSN 只有 24 bit，回绕时如何区分新包、重复包和越界包？</summary>

答：全部按模 2^24 运算，并规定一个 SQ 同时未确认的 PSN 不超过 2^23（C9-61, p.308），于是 PSN 空间被平分为有效区与无效区：有效区是以期望 PSN 结尾的 2^23 个 PSN，其中除期望 PSN 外都是重复区〔Vol1 Figure 88, p.309〕。旧连接残留的报文由 CM 通过 TimeWait 与随机起始 PSN 来规避〔Vol1 §9.7.1, p.308；§12.7.31, p.737〕。算法与 Verilog 片段见 [常数与公式](05-constants-and-formulas.md)。
</details>

<details><summary>Q27：重复请求如何处理？SEND/WRITE、READ、ATOMIC 有何不同？</summary>

答：处理重复请求期间响应方不更新期望 PSN（C9-103）。SEND/WRITE 的重复包不重复执行，只回 ACK，ACK 中 PSN 为最近完成的新请求（C9-105, p.338）；READ 的重复请求要**重新读取**并从重复请求的 PSN 开始回送响应（C9-107, p.340），期间内存若被改写，数据可能与第一次不同；ATOMIC 的重复请求不得重执行，须返回保存的原结果（o9-66, p.342），因此响应方要为未完成的原子操作保留结果缓存。详见 [响应方](../02-transport/07-responder.md)。
</details>

<details><summary>Q28：请求方收到 NAK 码为保留值的确认包怎么办？</summary>

答：视为畸形包静默丢弃（C9-120, p.349），之后可能因等不到确认而超时，再重传或停止该 SQ。
</details>

<details><summary>Q29：为什么同一个 QP 的报文一旦乱序，性能会急剧下降？</summary>

答：RC 响应方只接受 PSN 等于期望值的新请求。报文乱序到达时，先到的"超前"包落入无效区，响应方回 NAK（PSN Sequence Error），并丢弃后续超前的包，直到收到期望 PSN（C9-79、C9-80, p.325）；请求方随后从 NAK 指示的 PSN 起 Go-back-N 重传。于是一次乱序就会造成大量重传。这正是规范要求同一 QP 使用固定 UDP 源端口、让 ECMP 把它固定在一条路径上的原因〔Vol1 §A17.3.2.1, p.1934；§A17.9.4, p.1948〕。
</details>

## 5. Verbs 与 CM

<details><summary>Q30：AV 中"目的 L2 地址"设为 0xFFFFFFFFFFFF 是什么意思？</summary>

答：表示由 Verbs 实现（Create AH、Modify QP）自己完成 L3→L2 地址解析；否则调用者可以直接传入已解析好的 DMAC，实现原样使用——Linux rdma_cm 就是先解析再创建 AV 的〔Vol1 §A17.4.2 及脚注 62, p.1937〕。L3→L2 解析本身不在规范范围，通常借助 OS 的 ARP/ND 与路由表。
</details>

<details><summary>Q31：RoCEv2 中还有 QP0 吗？</summary>

答：没有。QP0 只用于子网管理器与 SMA 通信，与 RoCEv2 无关；目的 QP 为 0 的报文须静默丢弃（CA17-33, p.1943）。Get Special QP 只适用于 QP1（GSI），对其他类型须返回"Invalid Special QP Type"；C11-13 关于 QP0 的部分、o11-1、C17-24 都不适用〔Vol1 §A17.5.8, p.1942；§A17.7.5, p.1945〕。
</details>

<details><summary>Q32：CM 消息在 RoCEv2 中是怎么传输的？</summary>

答：RoCEv2 沿用第 12 章的 IB CM，CM 报文就是与目标连接同 IP 版本的普通 RoCEv2 报文〔Vol1 §A17.6.1, p.1943〕：MAD（256 字节，MgmtClass 0x07）装在 UD 报文里发往对端 QP1（GSI），DETH 中的 Q_Key 为知名值 0x80010000（C9-49, p.301）。使用 rdma_cm 时，REQ 私有数据按 Annex A11 格式携带源/目的 IP 与端口。详见 [CM 协议](../04-connection-management/01-cm-protocol.md)、[MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md)。
</details>

<details><summary>Q33：REQ 消息中的 LID 字段怎么处理？哪些 REJ/APR 码不能发送？</summary>

答：RoCEv2 端口间，REQ 的 Primary/Alternate Local/Remote Port LID 四个字段保留，接收方忽略且不得校验（CA17-34）；LAP 的备用 LID、SAP 的 Alternate Local Port LID 同样处理（CA17-36、CA17-38）。不得发送 REJ 原因码 13（Primary Remote Port LID rejected）、19（Alternate Remote Port LID rejected）（CA17-35），也不得发送 APR 状态码 7（CA17-37）。
</details>

<details><summary>Q34：SL 8–15 在 RoCEv2 中可以用吗？</summary>

答：不能，SL 8–15 保留〔Vol1 §A17.4.2, p.1938〕。创建/修改 AH、Modify QP、Modify EE 时使用保留 SL **可以**返回 Invalid Address Vector〔Vol1 §A17.5.3–A17.5.5, p.1941〕；RoCE v1 的 CA16-18 则写的是"必须"返回该错误。实际只用 0–7（对应 8 个以太网优先级）。
</details>

<details><summary>Q35：UD 接收缓冲为什么要预留 40 字节？IPv4 时 IP 头放在哪里？</summary>

答：IB 规定 UD 接收缓冲的前 40 字节留给 GRH〔Vol1 §11.4.1.2〕。RoCEv2 中这块区域放 IP 头：IPv6 头正好占满 40 字节；IPv4 头只有 20 字节，放在**偏移 20** 处（后半段），前 20 字节内容未定义〔Vol1 §A17.4.5.2, p.1939〕。UD 的 CQE 还要带上源 MAC 和"IPv4 / IPv6 / RoCE"报文类型标志〔Vol1 §A17.4.5.1〕，Linux 内核中对应 ib_wc 的 network_hdr_type 等字段〔业界〕。
</details>

<details><summary>Q36：rdma_cm 与 IB CM 是什么关系？</summary>

答：rdma_cm（内核 rdma_cm 与用户态 librdmacm）负责把 IP 地址/端口解析为 GID、路由和 DMAC，然后用 IB CM 的 REQ/REP/RTU 经 QP1 建立连接；REQ 私有数据前 36 字节按 Annex A11 放版本、IP 版本、源端口和源/目的 IP，Service ID 为 0x0000000001 + 协议号 + 端口〔Vol1 Table 596/597, p.1850–1852〕。Linux 的 RDMA_PS_TCP（0x0106）端口空间正好对应"0x01 前缀 + TCP 协议号 6"〔业界〕。详见 [IP CM 与 rdma_cm](../04-connection-management/02-ip-cm-and-rdma-cm.md)。
</details>

<details><summary>Q37：GID index 选错会出现什么现象？</summary>

答：常见三种：选到 IPv4/IPv6 类型不对的表项（源 IP 错误或协议版本不匹配）；选到 RoCE v1 类型表项（发出 0x8915 报文，v2-only 的对端会丢弃，CA17-41）；选到另一个 VLAN 接口上的 IP（VLAN/优先级错误，可能根本不可达）。表现通常是连接建立失败或 QP 以"重试超限"错误完成。排查时核对每个 GID index 的值与类型〔业界: show_gids、sysfs gid_attrs〕。
</details>

## 6. 拥塞控制与 QoS

<details><summary>Q38：RoCEv2 CNP 的 DestQP 填谁？发往哪里？</summary>

答：Figure 349 规定 DestQP"设为为之生成 CNP 的那个 QP 号"〔Vol1 p.1948〕，也就是**需要降速的那个 QP——被 ECN 标记报文的源 QP**，CNP 发往被标记报文的源端（CA17-44）。被标记报文的 BTH 里只有目的 QP，所以接收端要从自己的 QP 上下文（RC/UC/XRC 的远端 QPN）或 DETH 的 SrcQP（UD）得到源 QP 号——与 IB CC 中 CNP 的做法一致〔Vol1 Annex A10 §A10.3.2(d), p.1815〕。
</details>

<details><summary>Q39：RoCEv2 CNP 与 IB（Annex A10）CNP 有何不同？</summary>

答：触发条件：RoCEv2 由 IP 头 ECN = 11（CE）触发，IB 由 BTH 的 FECN 触发。OpCode：RoCEv2 为 0b1000_0001（0x81），IB 为 0b1000_0000（0x80）。格式：RoCEv2 CNP = MAC + IP + UDP + BTH + 16 字节保留（发送置 0）+ ICRC + FCS，BTH 中 PSN = 0、SE = 0、M = 0、P_Key 与被标记报文相同〔Vol1 Figure 349, p.1948〕；IB CNP 带 LRH（可有 GRH）、BTH 的 B（BECN）位置 1，PSN 为 0 且接收方忽略〔Vol1 §A10.3.2, p.1815–1816〕。IB 还可以把 BECN 捎带在 ACK 中，RoCEv2 不用这种方式。
</details>

<details><summary>Q40：IB 拥塞控制的 FECN/BECN 位在 RoCEv2 中还用吗？</summary>

答：不用。BTH 的 F（FECN）、B（BECN）位不使用，RoCEv2 端口接收时忽略（CA17-29, p.1940），Annex A10 整体对 RoCEv2 端口不适用〔Vol1 §A17.4.8〕。RoCEv2 改用 IP 头的 ECN，并"按 BTH FECN 的精神"解释 CE 标记〔Vol1 §A17.9.3, p.1947〕。另外 F/B 位所在字节本来就不计入 ICRC。
</details>

<details><summary>Q41：收到 CE 标记的报文，必须每个都回一个 CNP 吗？</summary>

答：不必。CA17-44 允许 HCA 对同一 QP 上的多个 ECN 标记报文只发一个 CNP。DCQCN 论文中接收端对每条流至多每 50 µs 发送一个 CNP〔论文: DCQCN, SIGCOMM 2015〕，商用网卡通常把这个间隔做成可配置参数〔业界，厂商相关〕。只有**有效**报文（如 ICRC 正确）才应触发 CNP。
</details>

<details><summary>Q42：DCQCN 是 RoCEv2 规范的一部分吗？</summary>

答：不是。A17 只规定了 RCM 的框架：收到 CE 回 CNP（CA17-44）、收到 CNP 降速（CA17-45）、经过一定时间/字节后提速（CA17-46，should），而降速幅度、提速时间、字节数和增幅的配置都"不在规范范围"。DCQCN 是 SIGCOMM 2015 论文提出、被主流网卡实现的具体算法。详见 [DCQCN](../05-congestion-control-and-qos/04-dcqcn.md)。
</details>

<details><summary>Q43：为什么同一个 QP 要用固定的 UDP 源端口？不同 QP 又该怎样？</summary>

答：数据中心路由器用包含 UDP 源端口的哈希做 ECMP 选路。同一 QP（有序报文序列）使用恒定源端口才能走同一条路径、保持顺序〔Vol1 §A17.3.2.1, p.1934；§A17.9.4, p.1948–1949〕；否则乱序会触发 NAK 与 Go-back-N 重传（见 Q29）。彼此没有顺序约束的报文（如不同 QP）则应使用不同源端口，以获得足够的哈希熵、均衡链路负载。Linux 由本端/远端 QPN 计算 Flow Label，再映射到 0xC000–0xFFFF 的源端口〔业界: rdma_calc_flow_label()、rdma_flow_label_to_udp_sport()〕。
</details>

<details><summary>Q44：SL、PCP、DSCP、Traffic Class 是什么关系？</summary>

答：AV 的 Traffic Class 写入 IP 头 DSCP（CA17-4、CA17-13）；AV 的 SL（0–7）映射为以太网优先级，体现在 VLAN 标签的 PCP 中〔Vol1 §A17.4.2, p.1938〕。交换机按"信任模式"用 PCP 或 DSCP 把报文分到优先级队列与 PFC 类别。跨三层时 VLAN 标签逐跳重建，PCP 不能端到端保持，所以通常按 DSCP 分类，并要求路由器把 RoCEv2 映射到下一子网启用 PFC 的优先级〔Vol1 §A17.9.1〕。常见做法是 RoCE 数据与 CNP 分别使用不同优先级（例如数据用优先级 3/DSCP 26、CNP 用优先级 6/DSCP 48，厂商相关，以文档为准）。
</details>

<details><summary>Q45：PFC 和 ECN 都要配吗？阈值之间有什么关系？</summary>

答：通常两者都配：ECN/CNP（如 DCQCN）负责在拥塞初期把速率降下来，PFC 作为最后防线防止缓冲溢出丢包。工程上要让 ECN 标记阈值（Kmin/Kmax）低于 PFC 的 XOFF 阈值，使降速先于暂停发生，并为每个无损优先级预留足够 headroom〔业界实践〕。规范层面二者都只是推荐/可选：无损是 should〔Vol1 §A17.9.1〕，RCM 是可选规范〔Vol1 §A17.9.3〕。详见 [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md)。
</details>

<details><summary>Q46：IB 的 QoS（Annex A13）对 RoCEv2 有用吗？</summary>

答：没有直接作用。A13 依赖 IB 链路层能力（VL、SL→VL 映射、VL 仲裁等），对 RoCEv2 端口不适用〔Vol1 §A17.4.9, p.1940〕。RoCEv2 的 QoS 使用以太网/IP 机制：ETS（802.1Qaz）分配带宽，DSCP / 802.1Q 标识流量，并由 AV 的 Traffic Class 控制〔Vol1 §A17.9.2, p.1946–1947〕。
</details>

## 7. 硬件实现

<details><summary>Q47：用 FPGA 实现 RoCEv2 需要实现 IB 物理层或链路层吗？</summary>

答：不需要。RoCE 用以太网替换了 IB 的链路层与物理层〔Vol1 §A17.2.2, p.1928〕，直接复用 FPGA 的以太网 MAC/PCS/RS-FEC 硬核和标准以太网光模块即可；VL 信用流控、LRH、VCRC、IB 链路训练和 Heartbeat 都不用做。需要自己实现的是 IP/UDP/BTH 头生成与解析、IPv4 头校验和、ICRC、传输层可靠性以及与 MAC 的 PFC 集成。详见 [IB 物理层对比](04-ib-physical-layer-vs-ethernet.md)、[FPGA 实现要点](../06-hardware-implementation/05-fpga-implementation-notes.md)。
</details>

<details><summary>Q48：接收侧哪些情况必须"静默丢弃"？哪些情况要回 NAK？</summary>

答：静默丢弃（不回 NAK、不产生 CQE、只计数）：不符合以太类型/IP/UDP/ICRC 规则的报文（CA17-24）；IP 版本与 Ethertype 不符（CA17-27）；DestQP = 0（CA17-33）；DGID 检查中 GID type 不符（CA17-41）；P_Key 不匹配（C9-43）；UD 的 Q_Key 不匹配（C10-14）；请求方收到含保留 NAK 码的确认（C9-120）；处理重复 SEND/WRITE 时出错（C9-106）；发出 PSN 序列错 NAK 后继续到达的超前包（§9.7.4.1.2, p.325）。回 NAK 的典型情况：RC 请求 PSN 首次越界（NAK 序列错，C9-79），以及 R_Key/长度/OpCode 等请求校验失败（Invalid Request、Remote Access Error 等）。
</details>

<details><summary>Q49：一个"最小可用"的 RoCEv2 RC 实现至少要支持哪些 OpCode？</summary>

答：要与 Linux/商用网卡的 RC 应用互通，至少需要：SEND First/Middle/Last/Only（含 with Immediate）、RDMA WRITE First/Middle/Last/Only（含 with Immediate）、RDMA READ Request 与 READ Response First/Middle/Last/Only、Acknowledge（含 NAK 与 RNR NAK）；支持 RCM 时再加 CNP（0x81）。原子操作对 HCA 是可选的〔Vol1 Table 376, p.1195〕。但要注意：按 C17-11，HCA 须能在任何 QP 上支持 UD、RC、UC；按 C17-12，支持某服务就要支持该服务定义的全部操作（原子操作除外）〔Vol1 p.1196〕。因此只实现 RC 子集的 FPGA 引擎严格说不构成合规 HCA，这是面向特定应用时常见的工程取舍（解读）。
</details>

<details><summary>Q50：RNIC 需要为每个 QP 维护哪些状态？</summary>

答：典型 QP 上下文包括：状态与服务类型、PMTU、P_Key、Q_Key（UD）、远端 QPN；地址信息（DMAC、源/目的 IP、DSCP、TTL/Hop Limit、Flow Label、UDP 源端口、VLAN/PCP）；请求方状态（下一 PSN、最早未确认 PSN、在途窗口、重试与 RNR 计数、传输定时器、ORD 与未完成 READ/Atomic 记录）；响应方状态（期望 PSN、MSN、信用、IRD 与 READ/Atomic 响应资源及重复请求缓存、多包消息的当前 VA/R_Key/剩余长度）；关联的 CQ、SRQ；拥塞控制状态（如 DCQCN 的速率与计时器）。规模估算见 [RNIC 架构](../06-hardware-implementation/01-rnic-architecture.md)，可靠性相关状态见 [可靠性引擎](../06-hardware-implementation/04-reliability-engine.md)。
</details>

<details><summary>Q51：为什么 FPGA 设计里必须自己计算 IPv4 头校验和？</summary>

答：以太网 MAC 只生成 FCS，不会计算 IP 头校验和；A17 虽未单列该字段，但 IPv4 要求头校验和有效〔外部: RFC 791〕，路由器和接收端会丢弃校验和错误的报文。它在 ICRC 中被屏蔽（CA17-22(c)），所以必须独立计算，而且每跳 TTL 改变后由路由器更新（解读）。
</details>

## 8. 运维排障

<details><summary>Q52：网卡 MTU 是 1500，为什么 ibv_devinfo 显示 active_mtu 为 1024？</summary>

答：Linux 用 `iboe_get_mtu()` 从网卡 MTU 中减去 96 字节的最大头部开销（GRH 40 + UDP 8 + BTH 12 + XRCETH 4 + AtomicETH 28 + ICRC 4），再取不超过结果的最大 IB MTU：1500 − 96 = 1404 → 1024〔业界: Linux include/rdma/ib_addr.h〕。要得到 4096，需要把网卡 MTU 设为 ≥ 4192（如 4200 或 9000），并保证全路径交换机/路由器 MTU 足够（IP 头 DF = 1，不会分片）。
</details>

<details><summary>Q53：用 tcpdump 抓不到 RoCE 报文，怎么办？</summary>

答：硬件 RNIC 绕过内核协议栈收发 RoCE 报文，主机上的 tcpdump 通常看不到。可选方法：交换机端口镜像抓包；使用网卡厂商提供的 RDMA 抓包功能（厂商相关）；或在测试环境使用 Soft-RoCE（rxe），其报文经过内核网络栈，可直接抓取〔业界〕。Wireshark 默认把 UDP 4791 解析为 RoCEv2（RRoCE），可显示 BTH 与扩展头。
</details>

<details><summary>Q54：应用大量出现 retry exceeded（IBV_WC_RETRY_EXC_ERR），常见原因有哪些？</summary>

答：含义是请求方的超时/序列错重试次数耗尽（C9-142、C9-143）。常见原因：网络丢包（PFC/ECN 未正确配置、缓冲溢出、链路误码导致 FCS 错误）；GID index、VLAN 或 IP 选错导致对端丢弃；路径 MTU 不足（DF = 1 被丢弃）；对端 QP 已进入错误态或被销毁；ACL/防火墙拦截 UDP 4791；Local ACK Timeout 过小而 PFC 暂停时间长。排查顺序：链路与 FEC 计数 → 交换机丢包/PFC/ECN 计数 → 网卡传输层计数（如 out_of_sequence、packet_seq_err、local_ack_timeout_err 等，名称厂商相关）→ GID/MTU/路由配置。详见 [排障指南](../07-engineering-practice/05-troubleshooting.md)。
</details>

<details><summary>Q55：出现 RNR retry exceeded（IBV_WC_RNR_RETRY_EXC_ERR）又是什么原因？</summary>

答：对端没有及时投递接收 WQE，持续回 RNR NAK，直到请求方的 RNR 重试次数耗尽（C9-132）。处理办法：提前、批量投递接收缓冲或使用 SRQ；必要时把 RNR Retry Count 设为 7（无限重试）；把最小 RNR 定时器设为合适的小值——注意编码 0 代表最长的 655.36 ms，不是"不等待"〔Vol1 Table 48, p.354〕。
</details>

<details><summary>Q56：如何判断网络中发生了 PFC 风暴，或者 ECN 根本没有生效？</summary>

答：PFC：观察网卡与交换机每个优先级的 PAUSE 帧收发计数与暂停时长；某端口持续发送暂停帧、暂停在网络中扩散，是风暴/死锁的典型迹象，交换机通常提供 PFC watchdog 一类保护功能（厂商相关）。ECN：看交换机的 ECN 标记计数，以及网卡的 CNP 收发计数（如 mlx5 的 np_ecn_marked_roce_packets、np_cnp_sent、rp_cnp_handled，厂商相关）。如果 PFC 计数持续上涨而 CNP 计数为 0，多半是交换机未开启 ECN、阈值过高，或网卡未启用 RCM。
</details>

<details><summary>Q57：RoCEv2 跨三层部署时要特别注意什么？</summary>

答：① 按 DSCP 分类并逐跳配置 PFC，路由器把 RoCEv2 映射到下一子网启用 PFC 的优先级〔Vol1 §A17.9.1〕；② 所有跳都开启 ECN 标记，且 CNP 走高优先级；③ 全路径 MTU 一致且足够（IPv4 DF = 1）；④ ECMP 哈希包含 UDP 源端口〔Vol1 §A17.9.4〕；⑤ AV 的 Hop Limit 足够大（Linux 通常取路由的默认 TTL，如 64，业界）；⑥ 中间设备不得改写 IP 地址或 UDP 端口（ICRC 会失败，见 Q19）。详见 [配置指南](../07-engineering-practice/02-configuration-guide.md)。
</details>

<details><summary>Q58：如何用最少的设备搭一个 RoCEv2 学习/调试环境？</summary>

答：两台装有普通以太网卡的 Linux 主机即可：用 `rdma link add rxe0 type rxe netdev <接口>` 创建 Soft-RoCE 设备，再用 rdma-core 自带的 ibv_rc_pingpong、perftest 的 ib_write_bw 等工具测试，并用 tcpdump/Wireshark 直接观察 RoCEv2 报文〔业界: Linux rdma_rxe、rdma-core、perftest〕。它也是硬件 RoCEv2 实现做互通测试时方便的参考端。详见 [Linux RDMA 软件栈](../07-engineering-practice/01-linux-rdma-stack.md)、[工具与测试](../07-engineering-practice/04-tools-and-testing.md)。
</details>

## 9. 要点回顾

- RoCEv2 = IB 传输层 + UDP/IP + 以太网；它替换了 IB 的物理层、链路层、网络层头和管理框架，保留传输层、Verbs 与 CM。
- 易混淆点：UDP 校验和为 0 是 should（由 ICRC 保护）；ICRC 屏蔽 TTL/TOS/Hop Limit/Flow Label/UDP 校验和但不屏蔽端口与地址；IPv6 Payload Length 等于 UDP Length（CA17-15 原文笔误）；RNR 编码 0 = 655.36 ms。
- v1 与 v2 的 GID 都在端口 GID 表里，决定封装的是 GID type；同一 IP 在 Linux 中常有 v1、v2 两个表项。
- CNP 的 DestQP 是需要降速的源 QP；RoCEv2 CNP OpCode 为 0x81，IB CNP 为 0x80；FECN/BECN 在 RoCEv2 中被忽略。
- 同一 QP 固定 UDP 源端口以保证 ECMP 下的顺序，不同 QP 使用不同源端口以获得熵。
- 接收侧大量错误都要"静默丢弃"，而 RC 的 PSN 越界、访问权限等错误则要回 NAK；硬件需把这两类路径分清。

## 参考

- 规范：Vol1 Annex A17（p.1928–1949）、Annex A16（p.1911–1927）、§7.8（p.218–231）、§9.2–9.9（p.256–449）、§11.4.1.2、§12.7（p.729–739）、§17.2.2（p.1194–1197）、Annex A10 §A10.3（p.1814–1816）、Annex A11（p.1848–1856）。
- 外部：RFC 768、791、3168、5040/5041/5044、6935、6936、8200；IEEE 802.1Qbb、802.1Qaz；IANA 端口注册表（UDP 4791）。
- 论文：DCQCN（SIGCOMM 2015）；IRN（SIGCOMM 2018）。
- 业界实践：Linux 内核 RDMA 子系统（ib_addr.h、ib_verbs.h、rdma_rxe）、rdma-core、perftest、Wireshark RRoCE 解析器；NVIDIA/Mellanox mlx5 计数器（厂商相关，以官方文档为准）。
