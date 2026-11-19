# 常数与公式速查（Constants and Formulas）

> **摘要**：把 RoCEv2 设计、调试、计算中反复用到的常数（Ethertype、端口、OpCode、Syndrome、NAK 码、RNR 定时器、MTU 编码、Q_Key/P_Key、CM 属性 ID……）、头部长度和公式（超时编码、PSN 模运算、长度字段、线速效率、PFC 量子与 headroom 估算、READ 深度）集中在一页，每项注明出处。文中 Python 代码均已在本地运行验证（RNR 与信用编码公式与规范表格逐项比对一致，ICRC 参数用规范样例验证）。
>
> **规范出处**：Vol1 §9.2–9.3（p.256–267）、Table 38（p.257–259）、Table 46–48（p.348–355）、Table 50（p.378–379）、§9.7.6（p.364）、§12.7（p.729–739）、§13.4（p.780–818）、§16.7（p.1182–1186）、§7.8（p.218–231）、Annex A17（p.1928–1949）
>
> **前置阅读**：[帧布局](../01-packet-format/01-frame-layout.md)、[BTH](../01-packet-format/03-bth.md)、[PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md)
>
> **关键词**：Ethertype, 4791, OpCode, Syndrome, NAK, RNR, PMTU, Ttr, PSN, PFC quanta, headroom, 线速效率

## 1. 以太网 / IP / UDP 常数

| 常数 | 值 | 说明 | 出处 |
|---|---|---|---|
| Ethertype IPv4 | 0x0800 | RoCEv2 over IPv4 | 〔Vol1 §A17.3.1.1, p.1932〕 |
| Ethertype IPv6 | 0x86DD | RoCEv2 over IPv6 | 〔Vol1 §A17.3.1.2, p.1933〕 |
| Ethertype RoCE v1 | 0x8915 | 后接 IB GRH | 〔Vol1 §A17.2.3 脚注 58, p.1929；CA16-2, p.1913〕 |
| VLAN TPID | 0x8100 | 802.1Q 标签：PCP(3) + DEI(1) + VID(12) | 〔外部: IEEE 802.1Q〕 |
| MAC 控制帧 Ethertype | 0x8808 | PAUSE opcode 0x0001；PFC opcode 0x0101 | 〔外部: IEEE 802.3 Clause 31、Annex 31B/31D；802.1Qbb〕 |
| PAUSE/PFC 目的 MAC | 01-80-C2-00-00-01 | MAC 控制帧组播地址 | 〔外部: IEEE 802.3〕 |
| LLDP Ethertype | 0x88CC | DCBX 的载体 | 〔外部: IEEE 802.1AB〕 |
| IPv4 Protocol / IPv6 Next Header | 0x11（17，UDP） | 不带 IPv6 扩展头 | CA17-10、CA17-16〔Vol1 p.1933–1934〕 |
| IPv4 IHL | 5 | 无 IP 选项 | CA17-3〔Vol1 p.1932〕 |
| IPv4 Flags | 010b | DF = 1，MF = 0 | CA17-7〔Vol1 p.1932〕 |
| IPv4 Fragment Offset | 0 | 不分片 | CA17-8〔Vol1 p.1933〕 |
| UDP 目的端口 | 4791 | 规范写"IANA 分配值"，脚注称分配后更新 | CA17-20〔Vol1 p.1934–1935〕；〔外部: IANA〕 |
| UDP 校验和 | 0 | should | 〔Vol1 §A17.3.2.4, p.1935〕 |
| UDP 源端口（Linux 惯例） | 0xC000–0xFFFF | 由 Flow Label 派生，Flow Label 为 0 时由本端/远端 QPN 计算 | 〔业界: Linux include/rdma/ib_verbs.h rdma_flow_label_to_udp_sport()〕 |
| IPv4 映射 GID | ::ffff:a.b.c.d | 高 80 bit 为 0，随后 16 bit 全 1，低 32 bit 为 IPv4 | 〔Vol1 §A17.4.1.1, p.1936〕 |
| AV"目的 L2 地址"特殊值 | 0xFFFFFFFFFFFF | 表示由 Create AH / Modify QP 的实现自行做 L3→L2 解析 | 〔Vol1 §A17.4.2, p.1937〕 |
| SL → 以太网优先级 | SL 0–7 → 优先级 0–7；SL 8–15 保留 | 优先级体现在 VLAN 标签 PCP 中 | 〔Vol1 §A17.4.2, p.1938〕 |
| GRH NxtHdr（RoCE v1） | 0x1B | 表示后接 IBA 传输头 | C9-44〔Vol1 p.298〕 |
| Reserved GID / Loopback GID | :: / ::1 | 不得用作 IBA 报文地址 | 〔Vol1 §4.1.1, p.161〕 |
| IB 默认 GID 前缀 | FE80::/64 | 链路本地，路由器不转发 | 〔Vol1 §4.1.1, p.160〕 |

### 1.1 ECN 码点与 TOS 字节

| ECN（2 bit） | 名称 | RoCEv2 用法 |
|---|---|---|
| 00 | Not-ECT | 不支持 RCM 的 HCA 必须使用（CA17-5、CA17-14） |
| 01 | ECT(1) | 支持 RCM 时可用（may）〔Vol1 §A17.3.1.1.3, p.1932〕 |
| 10 | ECT(0) | 同上 |
| 11 | CE | 网络设备标记拥塞；接收端据此生成 CNP（CA17-44） |

〔外部: RFC 3168；RFC 2474〕IPv4 TOS 字节与 IPv6 Traffic Class 字节布局相同：

```text
  7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
|          DSCP         |  ECN  |      TOS = (DSCP << 2) | ECN
+---+---+---+---+---+---+---+---+
例：DSCP 26 → TOS 0x68；DSCP 26 + ECT(0) → 0x6A；DSCP 48 → 0xC0
```

> [!NOTE]
> 规范要求 DSCP = AV 的 Traffic Class（CA17-4、CA17-13），但 Traffic Class 为 8 bit、DSCP 为 6 bit。常见实现把 Traffic Class 当作整个 TOS 字节、取高 6 bit 作 DSCP，ECN 位由拥塞控制逻辑决定〔业界，厂商相关〕。

## 2. IB 传输层与管理常数

| 常数 | 值 | 说明 | 出处 |
|---|---|---|---|
| BTH 长度 | 12 字节 | 所有 IBA 传输报文都有 | 〔Vol1 §9.2, p.256〕 |
| BTH TVer | 0x0 | 接收端不支持的版本则丢弃 | C9-5〔Vol1 p.261〕 |
| BTH F / B / Resv6 | 发送置 0 | 不计入 ICRC；RoCEv2 接收时忽略 F/B | C9-6〔Vol1 p.261〕；CA17-29〔Vol1 p.1940〕 |
| BTH Resv7 | 发送置 0 | **计入** ICRC | C9-7〔Vol1 p.262〕 |
| PadCnt | 0–3 | 负载补齐到 4 字节整数倍 | 〔Vol1 §9.2.5, p.260〕 |
| PSN | 24 bit，模 2^24 = 16,777,216 | — | 〔Vol1 §9.7.1, p.308〕 |
| 单个 SQ 最大未确认 PSN 数 | 2^23 = 8,388,608 | 用于区分重复包与乱序包 | C9-61〔Vol1 p.308〕 |
| QPN | 24 bit | — | 〔Vol1 §9.2.8, p.261〕 |
| QP0 | SMI | RoCEv2 端口不存在，DestQP = 0 的报文静默丢弃 | CA17-33〔Vol1 p.1943〕 |
| QP1 | GSI | CM 等 GMP 使用 | 〔Vol1 §17.2.7, p.1199〕 |
| 多播 DestQP | 0xFFFFFF | 只用于 UD 多播 | o10-35〔Vol1 p.500〕 |
| 默认 P_Key | 0xFFFF | 默认分区、完全成员 | C10-115〔Vol1 p.557〕 |
| 无效 P_Key | 低 15 bit 全 0（0x0000、0x8000） | — | C10-116〔Vol1 p.557〕 |
| P_Key 成员类型 | bit15 = 1 完全成员，0 受限成员 | 受限成员之间不能通信 | 〔Vol1 §10.9.1, p.556–557〕 |
| P_Key 表大小 | 1～65535 项 | — | C10-121〔Vol1 p.558〕 |
| QP1 知名 Q_Key | 0x80010000 | QP1 收到的 DETH Q_Key 与之比较 | C9-49〔Vol1 p.301〕 |
| WR 中 Q_Key 的最高位 | 1 = 改用 QP 上下文中的 Q_Key | 受控 Q_Key 机制 | C10-15〔Vol1 p.471〕 |
| DMALen 范围 | 0～2^31 字节 | RDMA 单次操作长度 | C9-9〔Vol1 p.264〕 |
| 包负载上限 | 4096 字节 | — | 〔Vol1 Glossary, p.92〕 |
| Retry Count | 3 bit | 超时 / PSN 序列错 NAK 的重传次数 | C9-142〔Vol1 p.364〕 |
| RNR Retry Count | 3 bit，7 = 无限重试 | — | C9-132〔Vol1 p.354〕 |
| Local ACK Timeout | 5 bit，0 = 禁用定时器 | Ttr = 4.096 µs × 2^n | C9-140〔Vol1 p.364〕 |
| 原子操作对齐 | 8 字节自然对齐，否则 Invalid Request NAK | 操作数 64 bit | 〔Vol1 §9.4.5, p.283〕 |
| MAD 负载 | 256 字节 = 24 字节公共头 + 232 字节数据 | — | C13-3〔Vol1 p.781〕；Table 121〔Vol1 p.782–783〕 |
| MAD BaseVersion | 1 | — | Table 121〔Vol1 p.782〕 |
| CM 管理类 MgmtClass | 0x07 | ComMgt | Table 122〔Vol1 p.784〕 |
| CM 方法 | ComMgtGet 0x01、ComMgtSet 0x02、ComMgtSend 0x03、ComMgtGetResp 0x81 | CM 消息用 Send | Table 371〔Vol1 p.1184〕 |
| CM 传输服务类型 | 0 RC、1 UC、2 RD、3 见扩展类型（1 = XRC） | REQ 字段 | 〔Vol1 §12.7.6, p.733；§12.7.41, p.739〕 |
| RoCE v1 默认 RespTimeValue / SubnetTimeout | 19 / 18 | 可被以太网管理实践修改 | CA16-9〔Vol1 p.1915〕 |

### 2.1 CM 属性 ID（Table 372, p.1184–1185）

| 属性 | ID | 对应消息 |
|---|---|---|
| ClassPortInfo | 0x0001 | — |
| ConnectRequest | 0x0010 | REQ |
| MsgRcptAck | 0x0011 | MRA |
| ConnectReject | 0x0012 | REJ |
| ConnectReply | 0x0013 | REP |
| ReadyToUse | 0x0014 | RTU |
| DisconnectRequest | 0x0015 | DREQ |
| DisconnectReply | 0x0016 | DREP |
| ServiceIDResReq | 0x0017 | SIDR_REQ |
| ServiceIDResReqResp | 0x0018 | SIDR_REP |
| LoadAlternatePath | 0x0019 | LAP |
| AlternatePathResponse | 0x001A | APR |
| SuggestAlternatePath | 0x001B | SAP |
| SuggestPathResponse | 0x001C | SPR |

### 2.2 RDMA IP CM Service（Annex A11）

| 常数 | 值 | 出处 |
|---|---|---|
| Service ID 格式 | 字节 0–3 = 0x00；字节 4 = 0x01；字节 5 = IP 协议号；字节 6–7 = 目的端口 | Table 597〔Vol1 p.1852〕 |
| Service ID 示例 | TCP 860（iSCSI/iSER）→ 0x000000000106035C；SCTP 2049（NFSv4-RDMA）→ 0x0000000001840801 | 〔Vol1 p.1852〕 |
| REQ 私有数据 | 字节 0–35：MajV/MinV(=0)、IPV(4 或 6)、源端口、源 IP(128 bit)、目的 IP(128 bit)；字节 36–91：消费者私有数据（56 字节） | Table 596〔Vol1 p.1850〕 |
| SIDR_REQ 私有数据 | 同上，消费者私有数据占字节 36–215 | CA11-5.1.1〔Vol1 p.1851〕 |
| IPv4 地址放置 | 只用低 32 bit，高 96 bit 置 0 | CA11-5〔Vol1 p.1851〕 |

### 2.3 MTU / PMTU 编码

| 编码 | MTU（负载字节） |
|---|---|
| 1 | 256 |
| 2 | 512 |
| 3 | 1024 |
| 4 | 2048 |
| 5 | 4096 |
| 0、6–15 | 保留（PortInfo）；PathRecord 中 0、6–63 保留 |

出处：PortInfo:NeighborMTU〔Vol1 p.906〕、PortInfo:MTUCap〔Vol1 p.909〕、PathRecord:Mtu〔Vol1 p.1040〕。libibverbs 的 `enum ibv_mtu` 也用 IBV_MTU_256 = 1 … IBV_MTU_4096 = 5〔业界〕。

## 3. OpCode 速查

高 3 bit：000 RC、001 UC、010 RD、011 UD、100 CNP、101 XRC、110/111 厂商自定义〔Vol1 Table 38, p.257–259〕。

### 3.1 RC（0x00–0x1F）

| Hex | 操作 | BTH 之后的头 |
|---|---|---|
| 0x00 | SEND First | PayLd |
| 0x01 | SEND Middle | PayLd |
| 0x02 | SEND Last | PayLd |
| 0x03 | SEND Last with Immediate | ImmDt, PayLd |
| 0x04 | SEND Only | PayLd |
| 0x05 | SEND Only with Immediate | ImmDt, PayLd |
| 0x06 | RDMA WRITE First | RETH, PayLd |
| 0x07 | RDMA WRITE Middle | PayLd |
| 0x08 | RDMA WRITE Last | PayLd |
| 0x09 | RDMA WRITE Last with Immediate | ImmDt, PayLd |
| 0x0A | RDMA WRITE Only | RETH, PayLd |
| 0x0B | RDMA WRITE Only with Immediate | RETH, ImmDt, PayLd |
| 0x0C | RDMA READ Request | RETH |
| 0x0D | RDMA READ Response First | AETH, PayLd |
| 0x0E | RDMA READ Response Middle | PayLd |
| 0x0F | RDMA READ Response Last | AETH, PayLd |
| 0x10 | RDMA READ Response Only | AETH, PayLd |
| 0x11 | Acknowledge | AETH |
| 0x12 | ATOMIC Acknowledge | AETH, AtomicAckETH |
| 0x13 | CmpSwap | AtomicETH |
| 0x14 | FetchAdd | AtomicETH |
| 0x15 | 保留 | — |
| 0x16 | SEND Last with Invalidate | IETH, PayLd |
| 0x17 | SEND Only with Invalidate | IETH, PayLd |
| 0x18–0x1F | 保留 | — |

### 3.2 其他服务类型

| 范围 | 服务 | 要点 |
|---|---|---|
| 0x20–0x2B | UC | 与 RC 的 0x00–0x0B 一一对应（SEND、RDMA WRITE）；0x2C–0x3F 保留 |
| 0x40–0x55 | RD | 请求带 RDETH + DETH，响应带 RDETH；0x55 = RESYNC（RDETH, DETH）；0x56–0x5F 保留 |
| 0x64 / 0x65 | UD | SEND Only（DETH, PayLd）/ SEND Only with Immediate（DETH, ImmDt, PayLd）；其余保留 |
| 0x80 | IB CNP | 基础规范 Table 38 中唯一定义的 CNP 码（Annex A10 使用） |
| **0x81** | **RoCEv2 CNP** | 0b1000_0001，BTH 后接 16 字节保留区〔Vol1 Figure 349, p.1948〕；在基础规范 Table 38 中 100_00001 原为保留值 |
| 0xA0–0xB7 | XRC | 编号规律同 RC（0xA0 = SEND First … 0xB7 = SEND Only with Invalidate）；请求报文在 BTH 后先带 XRCETH，READ 响应与 ACK（0xAD–0xB2）不带 XRCETH；0xB5 保留 |
| 0xC0–0xFF | 厂商自定义 | 双方明确支持前不应使用〔Vol1 §9.2.2, p.259–260〕 |

## 4. AETH Syndrome、NAK 码与定时器

### 4.1 Syndrome（Table 46, p.348）与 NAK 码（Table 47, p.349）

| Syndrome（hex） | bits[6:5] | 含义 | 低 5 bit |
|---|---|---|---|
| 0x00–0x1F | 00 | ACK | 端到端信用编码；0x1F 表示信用无效 |
| 0x20–0x3F | 01 | RNR NAK | RNR 定时器编码（§4.3） |
| 0x40–0x5F | 10 | 保留 | — |
| 0x60–0x7F | 11 | NAK | NAK 码（下表） |
| 0x80–0xFF | — | bit7 保留，HCA 必须置 0（C9-119.a1） | — |

| Syndrome | NAK 码 | 含义 |
|---|---|---|
| 0x60 | 0 | PSN Sequence Error（响应方在 BTH PSN 中填入自己的期望 PSN，C9-121） |
| 0x61 | 1 | Invalid Request |
| 0x62 | 2 | Remote Access Error |
| 0x63 | 3 | Remote Operational Error |
| 0x64 | 4 | Invalid RD Request |
| 0x65–0x7F | 5–31 | 保留；请求方收到含保留码的确认须视为畸形包静默丢弃（C9-120） |

### 4.2 端到端信用编码（Table 50, p.378–379）

| 码 | 信用 | 码 | 信用 | 码 | 信用 | 码 | 信用 |
|---|---|---|---|---|---|---|---|
| 0 | 0 | 8 | 16 | 16 | 256 | 24 | 4096 |
| 1 | 1 | 9 | 24 | 17 | 384 | 25 | 6144 |
| 2 | 2 | 10 | 32 | 18 | 512 | 26 | 8192 |
| 3 | 3 | 11 | 48 | 19 | 768 | 27 | 12288 |
| 4 | 4 | 12 | 64 | 20 | 1024 | 28 | 16384 |
| 5 | 6 | 13 | 96 | 21 | 1536 | 29 | 24576 |
| 6 | 8 | 14 | 128 | 22 | 2048 | 30 | 32768 |
| 7 | 12 | 15 | 192 | 23 | 3072 | 31 | 无效 |

LSN = MSN + 信用值；请求方可以发送 SSN ≤ LSN 的消耗接收 WQE 的请求〔Vol1 §9.7.7.2, p.379〕。

### 4.3 RNR NAK 定时器（Table 48, p.354–355）

| 编码 | 延迟 (ms) | 编码 | 延迟 (ms) | 编码 | 延迟 (ms) | 编码 | 延迟 (ms) |
|---|---|---|---|---|---|---|---|
| 00000 | **655.36** | 01000 | 0.16 | 10000 | 2.56 | 11000 | 40.96 |
| 00001 | 0.01 | 01001 | 0.24 | 10001 | 3.84 | 11001 | 61.44 |
| 00010 | 0.02 | 01010 | 0.32 | 10010 | 5.12 | 11010 | 81.92 |
| 00011 | 0.03 | 01011 | 0.48 | 10011 | 7.68 | 11011 | 122.88 |
| 00100 | 0.04 | 01100 | 0.64 | 10100 | 10.24 | 11100 | 163.84 |
| 00101 | 0.06 | 01101 | 0.96 | 10101 | 15.36 | 11101 | 245.76 |
| 00110 | 0.08 | 01110 | 1.28 | 10110 | 20.48 | 11110 | 327.68 |
| 00111 | 0.12 | 01111 | 1.92 | 10111 | 30.72 | 11111 | 491.52 |

> [!WARNING]
> 编码 00000 不是"0 等待"，而是最长的 655.36 ms。把 RNR 定时器"清零"会让请求方在 RNR 后等待半秒以上，是常见的性能陷阱。请求方须至少等待编码对应的时间再重试（C9-130）。

信用表与 RNR 表其实遵循同一规律（本文归纳，已用下列代码与两张表逐项核对）：编码 k ≥ 2 时数值 = (2 + (k & 1)) × 2^((k >> 1) − 1)，即 2、3、4、6、8、12、16…；RNR 的单位是 0.01 ms，且编码 0 等价于 k = 32（65536 × 0.01 ms）。

```python
def e2e_credits(code: int):
    """Table 50：AETH 低 5 bit → 端到端信用数；31 = 无效"""
    if code == 31:
        return None
    if code < 2:
        return code
    return (2 + (code & 1)) << ((code >> 1) - 1)


def rnr_delay_ms(code: int) -> float:
    """Table 48：RNR NAK 定时器编码 → 最短等待时间（ms）"""
    k = 32 if code == 0 else code
    return 0.01 * ((2 + (k & 1)) << ((k >> 1) - 1)) if k >= 2 else 0.01 * k


assert [e2e_credits(c) for c in (0, 1, 5, 15, 30, 31)] == [0, 1, 6, 192, 32768, None]
assert abs(rnr_delay_ms(0) - 655.36) < 1e-9 and abs(rnr_delay_ms(0b11111) - 491.52) < 1e-9
```

## 5. 头部长度

| 头 / 字段 | 长度（字节） | 出现条件 | 出处 |
|---|---|---|---|
| 以太网 MAC 头 | 14 | 每帧（DA 6 + SA 6 + Type 2） | 〔外部: IEEE 802.3〕 |
| 802.1Q VLAN 标签 | 4 | 使用 PCP 时 | 〔外部: IEEE 802.1Q〕 |
| IPv4 头 | 20 | IHL = 5 | CA17-3 |
| IPv6 头 | 40 | 无扩展头 | CA17-16 |
| UDP 头 | 8 | 每个 RoCEv2 报文 | 〔Vol1 §A17.3.2, p.1934〕 |
| BTH | 12 | 每个 IBA 传输报文 | 〔Vol1 §5.2.3, p.172〕 |
| RDETH | 4 | RD | 〔Vol1 §5.2.4, p.174〕 |
| DETH | 8 | UD、RD | 〔Vol1 §5.2.5, p.174〕 |
| XRCETH | 4 | XRC 请求 | 〔Vol1 §9.3.8, p.267〕 |
| RETH | 16 | WRITE First/Only、READ Request | 〔Vol1 §5.2.6, p.175〕 |
| AtomicETH | 28 | CmpSwap、FetchAdd | 〔Vol1 §5.2.7, p.176〕 |
| AETH | 4 | ACK、Atomic ACK、READ 响应 First/Last/Only | 〔Vol1 §5.2.9, p.177〕 |
| AtomicAckETH | 8 | Atomic ACK | 〔Vol1 §5.2.10, p.177〕 |
| ImmDt | 4 | 带立即数的 SEND/WRITE | 〔Vol1 §5.2.11, p.178〕 |
| IETH | 4 | SEND with Invalidate | 〔Vol1 §5.2.12, p.178〕 |
| RoCEv2 CNP 保留区 | 16 | CNP | 〔Vol1 Figure 349, p.1948〕 |
| ICRC | 4 | 每个 IBA 传输报文 | 〔Vol1 §5.2.14, p.179〕 |
| FCS | 4 | 每帧 | 〔外部: IEEE 802.3〕 |
| 前导码 + SFD | 8 | 线上开销 | 〔外部: IEEE 802.3〕 |
| 最小帧间隙 IFG | 12 | 线上开销 | 〔外部: IEEE 802.3〕 |
| 最小帧长 | 64 | 含 FCS，不足由 MAC 填充 | 〔外部: IEEE 802.3〕 |
| 对照：LRH / GRH / VCRC | 8 / 40 / 2 | 仅 IB / RoCE v1（GRH） | 〔Vol1 §5.2.1、§5.2.2、§5.2.15〕 |

### 5.1 典型报文的总长（计算，IPv4、无 VLAN）

| 报文 | 传输层头 | 帧长（含 FCS） | 线上占用（+20） | IPv6 帧长 |
|---|---|---|---|---|
| ACK | BTH + AETH = 16 | 66 | 86 | 86 |
| CNP | BTH + 16 保留 = 28 | 78 | 98 | 98 |
| RDMA READ Request | BTH + RETH = 28 | 78 | 98 | 98 |
| CmpSwap / FetchAdd | BTH + AtomicETH = 40 | 90 | 110 | 110 |
| ATOMIC Acknowledge | BTH + AETH + AtomicAckETH = 24 | 74 | 94 | 94 |
| UD SEND Only（0 负载） | BTH + DETH = 20 | 70 | 90 | 90 |
| 数据包固定开销（Middle） | BTH = 12 | 62 + 负载 + pad | 82 + 负载 + pad | 82 + 负载 + pad |

> 注：IPv6 帧比 IPv4 多 20 字节；上表 IPv6 列为帧长，线上占用再加 20。

## 6. 长度字段计算

```text
pad              = (4 − L_payload mod 4) mod 4              → BTH.PadCnt
L_transport      = 12 (BTH) + Σ 扩展头（由 OpCode 决定）
UDP Length       = 8 + L_transport + L_payload + pad + 4      （CA17-21）
IPv4 Total Length = 20 + UDP Length                            （CA17-6）
IPv6 Payload Length = UDP Length                               （CA17-15，见下方警告）
以太网帧长（含 FCS）= 14 (+4 VLAN) + IP 头(20/40) + UDP Length + 4
线上占用          = max(帧长, 64) + 8 + 12
```

> [!WARNING]
> CA17-15 原文写"IPv6 Payload Length 从 IPv6 头之后第一个字节（即 BTH）算到 ICRC（含）"〔Vol1 §A17.3.1.2.3, p.1934〕，括号中的"即 BTH"是规范笔误：RoCEv2 中 IPv6 头之后是 **UDP 头**。按 IPv6 定义（RFC 8200）与实际实现〔业界: Linux rxe prepare_ipv6_hdr()〕，**IPv6 Payload Length = UDP 头(8) + BTH + 扩展头 + 负载 + pad + ICRC(4) = UDP Length**。

```python
def rocev2_lengths(payload: int, ext_hdr: int = 0, ipv6: bool = False, vlan: bool = False) -> dict:
    """ext_hdr：BTH 之后扩展头的总字节数（如 RETH=16、AETH=4）"""
    pad = (4 - payload % 4) % 4
    udp_len = 8 + 12 + ext_hdr + payload + pad + 4           # CA17-21
    ip_len = udp_len if ipv6 else 20 + udp_len               # IPv6 Payload Length / IPv4 Total Length
    frame = 14 + (4 if vlan else 0) + (40 if ipv6 else 20) + udp_len + 4   # 含 FCS
    wire = max(frame, 64) + 8 + 12                           # 前导码/SFD + IFG
    return dict(pad=pad, udp_len=udp_len, ip_len=ip_len, frame=frame, wire=wire)


# SEND Only 1001 字节：pad=3，UDP Length=1028，IPv4 Total Length=1048，帧长 1066
assert rocev2_lengths(1001) == dict(pad=3, udp_len=1028, ip_len=1048, frame=1066, wire=1086)
```

**分段与包数**：长度为 L 的 SEND/WRITE 消息，包数 N = max(1, ⌈L / PMTU⌉)；除最后（或唯一）包外每个包负载都等于 PMTU；零长度消息用 Only 类 OpCode〔Vol1 §9.4, p.268–281〕。例：RDMA WRITE 10000 字节、PMTU 4096 → First（RETH + 4096）、Middle（4096）、Last（1808）。

**以太网 MTU 与 RoCE MTU**：IP 层 MTU（网卡 MTU）须 ≥ PMTU + IP 头 + 8 + 最大传输层头 + 4。Linux 用保守常数：`iboe_get_mtu()` 从网卡 MTU 中减去 96 字节（GRH 40 + UDP 8 + BTH 12 + XRCETH 4 + AtomicETH 28 + ICRC 4）后，取不超过结果的最大 IB MTU，因此网卡 MTU 1500 → RoCE MTU 1024，网卡 MTU ≥ 4192 → 4096〔业界: Linux include/rdma/ib_addr.h〕。

## 7. 定时器与超时公式

### 7.1 通用编码：T = 4.096 µs × 2^x

| 字段 | 含义 | 出处 |
|---|---|---|
| Local ACK Timeout（QP 属性 / CM REQ 字段） | RC 传输定时器 Ttr；CM 中由 REQ 发送方按 2 × PacketLifeTime + 本端 CA 的 ACK 延迟计算 | C9-140〔Vol1 p.364〕；〔Vol1 §12.7.34, p.737〕 |
| Remote / Local CM Response Timeout | 对端回应 CM 消息的时限 / 含往返飞行时间的等待时限 | 〔Vol1 §12.7.4–12.7.5, p.733〕 |
| Service Timeout（MRA） | 发出 MRA 后完成 REP/RTU/APR/REJ 的最长时间 | 〔Vol1 §12.7.32, p.737〕 |
| Target ACK Delay | 目标 CA 收到消息到发出 ACK/NAK 的最大间隔 | 〔Vol1 §12.7.33, p.737〕 |
| PacketLifeTime（PathRecord） | 报文在网络中可能存在的最长时间 | 〔Vol1 §12.7.34, p.737；§12.9.8.4, p.757〕 |
| SubnetTimeout / RespTimeValue | MAD 传播时间上界 / MAD 响应时间上界；x = 20 ≈ 4.3 s | 〔Vol1 §13.4.6.2, p.791–792〕 |

### 7.2 传输定时器 Ttr = 4.096 µs × 2^n（C9-140, p.364）

超时须在 Ttr ≤ To ≤ 4·Ttr 之间被检测到（C9-141）。

| n | Ttr | n | Ttr | n | Ttr | n | Ttr |
|---|---|---|---|---|---|---|---|
| 0 | **禁用** | 8 | 1.049 ms | 16 | 268.4 ms | 24 | 68.72 s |
| 1 | 8.192 µs | 9 | 2.097 ms | 17 | 536.9 ms | 25 | 137.4 s |
| 2 | 16.38 µs | 10 | 4.194 ms | 18 | 1.074 s | 26 | 274.9 s |
| 3 | 32.77 µs | 11 | 8.389 ms | 19 | 2.147 s | 27 | 549.8 s |
| 4 | 65.54 µs | 12 | 16.78 ms | 20 | 4.295 s | 28 | 1099.5 s |
| 5 | 131.1 µs | 13 | 33.55 ms | 21 | 8.590 s | 29 | 2199.0 s |
| 6 | 262.1 µs | 14 | 67.11 ms | 22 | 17.18 s | 30 | 4398.0 s |
| 7 | 524.3 µs | 15 | 134.2 ms | 23 | 34.36 s | 31 | 8796.1 s |

> [!NOTE]
> Local ACK Timeout 的最小非零值由 CA 厂商定义；若 QP 中配置的非零值小于该最小值，CA 可使用自己的最小值（C9-140）。

### 7.3 推导公式

```text
Local ACK Timeout 选取：  n = 最小的 n，使 4.096 µs × 2^n ≥ 2 × PacketLifeTime + ACK 延迟   （§12.7.34）
TimeWait：               2 × PacketLifeTime + 对端 ACK 延迟                                 （§12.9.8.4, p.757）
REQ 视为丢失：            2 × PacketLifeTime + Remote CM Response Timeout                    （§12.9.8.6, p.759）
收到 MRA 后最早重发：      Service Timeout + PacketLifeTime                                  （§12.9.8.5, p.759）
MAD 请求-响应等待上界：    4.096 µs × (2^CommTimeValueOut + 2^CommTimeValueIn + 2^RespTimeValue)  （§13.4.6.3, p.793）
   例：RoCE 默认 SubnetTimeout=18、RespTimeValue=19 → 4.096 µs × (2^18 + 2^18 + 2^19) ≈ 4.29 s
重试耗尽前的大致时间（本文估算，实现相关）：(Retry Count + 1) × To，To ∈ [Ttr, 4·Ttr]
```

```python
def ib_time_us(x: int) -> float:
    """IB 超时字段通用编码：4.096 µs × 2^x"""
    return 4.096 * (1 << x)


def local_ack_timeout_for(min_us: float) -> int:
    """返回使 Ttr ≥ min_us 的最小 Local ACK Timeout（1..31）"""
    for n in range(1, 32):
        if ib_time_us(n) >= min_us:
            return n
    raise ValueError("超出 5 bit 编码范围")


assert local_ack_timeout_for(1100) == 9          # 2097.152 µs；n=8 只有 1048.576 µs
assert abs(ib_time_us(14) - 67108.864) < 1e-6     # n=14 ≈ 67.1 ms
```

## 8. PSN 运算

```text
next_PSN = (PSN + 1) mod 2^24                       （SEND、WRITE、ATOMIC 每包）
RDMA READ 之后的下一请求：next_PSN = (PSN + n) mod 2^24，n = max(1, ⌈DMALen / PMTU⌉)
                                                    （Table 42, p.314；零长度 READ 回 1 个 Response Only, p.281）
d = (a − b) mod 2^24；"a 在 b 之后" ⟺ 0 < d < 2^23   （同一 SQ 未确认 PSN ≤ 2^23，C9-61）
响应方：d = (PSN − ePSN) mod 2^24
        d = 0                → 期望的新请求
        2^23 < d < 2^24      → 重复区（ePSN 之前的 2^23 − 1 个 PSN）
        1 ≤ d ≤ 2^23         → 无效区（RC 回 NAK：PSN Sequence Error）
```

有效区 = 以 ePSN 结尾的 2^23 个 PSN，重复区 = 有效区去掉 ePSN〔Vol1 §9.7.1, Figure 88, p.309〕；恰好相差 2^23 的边界按图归入无效区（本文解读）。

```python
PSN_MOD = 1 << 24
HALF = 1 << 23


def psn_add(psn: int, n: int = 1) -> int:
    return (psn + n) % PSN_MOD


def psn_diff(a: int, b: int) -> int:
    """(a - b) mod 2^24"""
    return (a - b) % PSN_MOD


def psn_after(a: int, b: int) -> bool:
    """a 在 b 之后（考虑回绕）"""
    d = psn_diff(a, b)
    return 0 < d < HALF


def classify_request(psn: int, epsn: int) -> str:
    d = psn_diff(psn, epsn)
    if d == 0:
        return "expected"
    if d > HALF:
        return "duplicate"
    return "invalid"


def read_resp_packets(dma_len: int, pmtu: int) -> int:
    return max(1, -(-dma_len // pmtu))


assert psn_add(0xFFFFFF) == 0 and psn_after(0x000002, 0xFFFFFE)
assert classify_request(99, 100) == "duplicate" and classify_request(101, 100) == "invalid"
assert read_resp_packets(0, 4096) == 1 and read_resp_packets(10000, 4096) == 3
```

「硬件实现要点」24 bit 比较只需一个减法器取最高位：

```verilog
// a 在 b 之后：0 < (a - b) mod 2^24 < 2^23
wire [23:0] psn_d     = psn_a - psn_b;           // 24 bit 减法自然取模
wire        psn_after = (psn_d != 24'd0) && (psn_d[23] == 1'b0);
wire        psn_dup   = (psn_d != 24'd0) && (psn_d[23] == 1'b1) && (psn_d != 24'h800000);
```

（`psn_dup` 以 psn_a 为收到的 PSN、psn_b 为 ePSN，对应上面的重复区判定。）

## 9. 线速效率与包率

```text
线上占用 W = max(帧长, 64) + 20          （前导码/SFD 8 + IFG 12）
负载效率 η = L_payload / W
有效带宽 = 线速 × η
包率 PPS = 线速 / (8 × W)
```

| PMTU | 报文 | IPv4 线上占用 | IPv4 效率 | IPv6 线上占用 | IPv6 效率 |
|---|---|---|---|---|---|
| 256 | Middle（仅 BTH） | 338 | 75.74% | 358 | 71.51% |
| 1024 | Middle | 1106 | 92.59% | 1126 | 90.94% |
| 1024 | WRITE First/Only（+RETH） | 1122 | 91.27% | 1142 | 89.67% |
| 2048 | Middle | 2130 | 96.15% | 2150 | 95.26% |
| 4096 | Middle | 4178 | 98.04% | 4198 | 97.57% |
| 4096 | WRITE First/Only（+RETH） | 4194 | 97.66% | 4214 | 97.20% |
| 4096 | READ Resp First/Last/Only（+AETH） | 4182 | 97.94% | 4202 | 97.48% |

（无 VLAN；带 VLAN 时每帧再加 4 字节，效率约降 0.1 个百分点。）

| 线速 | 4096 负载 Middle（4178 B） | 1024 负载 Middle（1106 B） | 64 字节最小帧（84 B） |
|---|---|---|---|
| 25 Gb/s | 0.748 Mpps / 24.5 Gb/s 负载 | 2.83 Mpps / 23.2 Gb/s | 37.2 Mpps |
| 100 Gb/s | 2.99 Mpps / 98.0 Gb/s | 11.30 Mpps / 92.6 Gb/s | 148.8 Mpps |
| 400 Gb/s | 11.97 Mpps / 392.2 Gb/s | 45.21 Mpps / 370.3 Gb/s | 595.2 Mpps |

> [!TIP]
> 以 IPv4 为例，RoCE MTU 从 1024 提到 4096，100GbE 上的负载效率约从 92.6% 升到 98.0%，包率降为约 1/4，对网卡与交换机的包处理压力影响更大。反向 ACK（86 B）在 100GbE 上最高约 145 Mpps，ACK 合并策略决定了反向开销。

## 10. 以太网 / DCB / 拥塞相关公式

### 10.1 PFC / PAUSE 时间量子

1 quanta = 512 bit 时间；暂停时间 = quanta × 512 / 线速；PFC 帧中每个优先级的时间字段为 16 bit（最大 65535 quanta）〔外部: IEEE 802.3 Annex 31B/31D、802.1Qbb〕。

| 线速 | 1 quanta | 65535 quanta |
|---|---|---|
| 25 Gb/s | 20.48 ns | 1.342 ms |
| 100 Gb/s | 5.12 ns | 335.5 µs |
| 400 Gb/s | 1.28 ns | 83.9 µs |

### 10.2 PFC headroom 估算（工程估算，非规范）

从接收端决定发出 PAUSE 到对端真正停发，期间仍会涌入数据，需要每个无损优先级预留 headroom：

```text
H ≈ R/8 × (2 × L × t_p + T_resp) + MTU_frame(对端已开始发送的帧) + MTU_frame(本端 PAUSE 前正在发送的帧)
   R：线速；L：线缆长度；t_p：传播时延（约 5 ns/m）；
   T_resp：对端 MAC/PHY 处理与响应时间（含 FEC、交换芯片流水线，厂商相关）
例：100 Gb/s、100 m、T_resp 取 1 µs、帧 4200 B → 12.5 KB/µs × 2 µs + 8.4 KB ≈ 33.4 KB（每端口每优先级）
```

```python
def pfc_headroom_bytes(rate_gbps, cable_m, mtu_frame, t_resp_us, t_prop_ns_per_m=5.0):
    """粗略估算；实际以交换芯片厂商的 headroom 计算器为准"""
    delay_s = 2 * cable_m * t_prop_ns_per_m * 1e-9 + t_resp_us * 1e-6
    return rate_gbps * 1e9 * delay_s / 8 + 2 * mtu_frame


assert pfc_headroom_bytes(100, 100, 4200, 1.0) == 33400.0
```

详细推导与厂商差异见 [无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md)。

### 10.3 ECN 标记概率（RED/WRED 曲线）

```text
p(q) = 0                                  , q ≤ Kmin
p(q) = Pmax × (q − Kmin) / (Kmax − Kmin)  , Kmin < q ≤ Kmax
p(q) = 1                                  , q > Kmax
```

〔外部: RED（Floyd & Jacobson, 1993）；论文: DCQCN, SIGCOMM 2015〕阈值取值属工程调优，见 [ECN 与 CNP](../05-congestion-control-and-qos/03-ecn-and-cnp.md)、[DCQCN](../05-congestion-control-and-qos/04-dcqcn.md)。

### 10.4 BDP 与 READ 深度

```text
BDP = R × RTT / 8                         例：100 Gb/s × 10 µs = 125 KB ≈ 30 个 4 KB 包
跑满带宽所需的未完成 READ 数（ORD）≥ R × RTT / (8 × S)       S：单个 READ 大小
ORD 受限时的 READ 吞吐上限 ≈ ORD × S × 8 / RTT
例：100 Gb/s、RTT 10 µs、S = 4 KB → 需要 ORD ≥ 31；若 ORD = 16，READ 吞吐约 52 Gb/s
```

（本文推导；ORD/IRD 的协商见 CM 的 Initiator Depth / Responder Resources〔Vol1 §12.7.29–12.7.30, p.736〕。）

## 11. ICRC 参数速查

| 参数 | 值 | 出处 |
|---|---|---|
| 多项式 | 0x04C11DB7（与以太网 CRC-32 相同；反射形式 0xEDB88320） | 〔Vol1 §7.8.1, p.219〕 |
| 初值 | 0xFFFFFFFF | 〔Vol1 §7.8.1, p.219〕 |
| 输出 | 结果取反 | 〔Vol1 §7.8.1, p.219〕 |
| 比特序 | 按字节顺序输入，每字节最低位先进入（与以太网 CRC 一致） | 〔Vol1 §7.8.1, p.219〕 |
| RoCEv2 前缀 | 先输入 64 bit 全 1 | CA17-22(a)〔Vol1 p.1935〕 |
| 覆盖范围 | IP 头首字节 → ICRC 前最后一个字节（含 UDP 头、BTH、扩展头、负载、pad） | CA17-22(b) |
| 以 1 替换的字段 | IPv4：TOS、TTL、Header Checksum；IPv6：Traffic Class、Flow Label、Hop Limit；UDP Checksum；BTH 第 4 字节（F/B/Resv6，即 Resv8a） | CA17-22(c)(d)；〔Vol1 §7.8.1, p.218–219〕 |
| 规范样例 | Table 28：0x9625B75A；Table 33：0xB67D1ED1 | 〔Vol1 §7.8.4, p.226、p.230〕 |
| VCRC（对照） | 16 bit，多项式 0x100B，初值 0xFFFF | 〔Vol1 §7.8.2、§7.8.4.2, p.220–222〕 |

规范中的参数恰好就是 zlib/以太网 CRC-32，因此软件上可直接用 `zlib.crc32` 计算；线上 4 字节的顺序等于 CRC 值的**小端**字节序（与以太网 FCS 相同）。下列代码用 §7.8.4 Table 28 的掩码字节流验证：

```python
import struct
import zlib


def icrc_wire_bytes(masked_stream: bytes) -> bytes:
    """masked_stream：已按规则把可变字段替换为 0xFF 的字节流（RoCEv2 需含开头 8 字节 0xFF）"""
    return struct.pack('<I', zlib.crc32(masked_stream))


TABLE28 = bytes.fromhex(
    "F012375C000E17D20A202487FF87B1B3000DEC2A01710A1C015D400238F27A05"
    "0000000EBB884D85FD5CFBA4728BC0690ED40000")
assert len(TABLE28) == 52
assert icrc_wire_bytes(TABLE28).hex() == "9625b75a"   # 规范：Generated ICRC is 0x9625B75A
```

完整的 RoCEv2 掩码构造与硬件并行 CRC 实现见 [ICRC](../01-packet-format/06-icrc.md)。

## 12. 要点回顾

- 封装常数：0x0800/0x86DD（RoCEv2）、0x8915（RoCE v1）、UDP 4791、IP 协议 0x11、IHL 5、DF = 1；UDP 校验和 0、源端口按 QP 固定都是 should。
- 传输常数：TVer = 0、PSN 24 bit 且未确认数 ≤ 2^23、默认 P_Key 0xFFFF、QP1 Q_Key 0x80010000、多播 QPN 0xFFFFFF、DMALen ≤ 2^31。
- OpCode：RC 0x00–0x17、UC 0x20–0x2B、UD 0x64/0x65、IB CNP 0x80、**RoCEv2 CNP 0x81**、XRC 0xA0–0xB7。
- AETH Syndrome：0x00–0x1F ACK（0x1F = 无信用）、0x20–0x3F RNR NAK、0x60–0x64 NAK；RNR 编码 0 = 655.36 ms 而非 0。
- 所有 IB 超时都按 4.096 µs × 2^n 编码；Ttr 的 n = 0 表示禁用，n = 14 ≈ 67 ms。
- 长度：UDP Length = 8 + 传输头 + 负载 + pad + 4；IPv4 Total Length = 20 + UDP Length；IPv6 Payload Length = UDP Length（CA17-15 原文"即 BTH"为笔误）。
- 4 KB PMTU 下 IPv4 RoCEv2 负载效率约 98%，1 KB 时约 92.6%；网卡 MTU 1500 时 Linux 只会给出 1024 的 RoCE MTU。
- ICRC 参数与以太网 CRC-32 相同，可用 `zlib.crc32` 做参考模型，线上字节为小端序。

## 自测

<details><summary>Q1：IPv6 RoCEv2 RDMA WRITE Only 报文携带 4096 字节负载，UDP Length、IPv6 Payload Length 和以太网帧长（无 VLAN、含 FCS）各是多少？</summary>

答：UDP Length = 8 + 12 + 16 + 4096 + 0 + 4 = 4136；IPv6 Payload Length = UDP Length = 4136（不是规范括号里暗示的 4128）；帧长 = 14 + 40 + 4136 + 4 = 4194 字节。
</details>

<details><summary>Q2：响应方 ePSN = 0x000005，先后收到 PSN = 0xFFFFFE 和 PSN = 0x000010 的请求，分别如何归类？</summary>

答：(0xFFFFFE − 0x000005) mod 2^24 = 0xFFFFF9 > 2^23，属重复区，按重复请求处理；(0x000010 − 0x000005) = 0xB，落在 1～2^23，属无效区（超前），RC 回 NAK（PSN Sequence Error），BTH PSN 填 ePSN（C9-121）。
</details>

<details><summary>Q3：希望传输定时器不小于 1 ms，Local ACK Timeout 至少设多少？对应的超时检测窗口是多少？</summary>

答：4.096 µs × 2^8 = 1.049 ms ≥ 1 ms，故 n = 8；超时会在 Ttr～4·Ttr，即约 1.05～4.19 ms 之间被检测到（C9-141）。
</details>

## 参考

- 规范：Vol1 §4.1.1（p.160–161）、§7.8（p.218–231）、§9.2–9.3（p.256–267）、§9.4（p.267–291）、§9.7（p.304–381）、§10.2.5（p.470–471）、§10.9（p.556–563）、§12.7（p.729–739）、§12.9.8（p.756–761）、§13.4（p.780–818）、§16.7（p.1182–1186）、PortInfo（p.906–909）、PathRecord（p.1040）、Annex A11（p.1848–1856）、Annex A16（p.1911–1927）、Annex A17（p.1928–1949）。
- 外部：IEEE 802.3（MAC 控制帧、PAUSE/PFC、帧格式）、IEEE 802.1Q / 802.1Qbb / 802.1AB；RFC 768、791、2474、3168、8200；IANA 端口注册表（UDP 4791）；RED（Floyd & Jacobson, 1993）。
- 论文：DCQCN（SIGCOMM 2015）。
- 业界实践：Linux include/rdma/ib_addr.h（iboe_get_mtu）、include/rdma/ib_verbs.h（rdma_flow_label_to_udp_sport）、drivers/infiniband/sw/rxe/rxe_net.c；libibverbs `enum ibv_mtu`。
