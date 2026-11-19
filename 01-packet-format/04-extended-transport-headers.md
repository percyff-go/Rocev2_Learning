# 扩展传输头详解（Extended Transport Headers）

> **摘要**：BTH 之后可能跟随 0 到多个扩展传输头（Extended Transport Header, ETH），它们由 OpCode 决定、按固定次序排列，携带远端地址与密钥（RETH、AtomicETH）、确认与流控信息（AETH）、原子操作结果（AtomicAckETH）、立即数（ImmDt）、失效密钥（IETH）以及数据报寻址信息（DETH、XRCETH、RDETH）。本文给出每个头的位布局、字段语义、出现的 OpCode，以及 AETH 的全部编码表（Syndrome、NAK 码、RNR 定时器、信用）。
>
> **规范出处**：Vol1 §5.2.4–§5.2.12（p.174–178）；§9.3（p.262–267）；§9.4.1.1（p.272–274）；§9.4.5（p.283–290）；§9.6.1.3–§9.6.1.5（p.300–302）；§9.7.5.2（p.348–355）；§9.7.7.2（p.371–381）；§10.2.5（p.470–471）
>
> **前置阅读**：[03-bth.md](03-bth.md)、[05-opcodes.md](05-opcodes.md)
>
> **关键词**：RETH, AtomicETH, AETH, Syndrome, NAK, RNR NAK, Credit, MSN, AtomicAckETH, ImmDt, IETH, DETH, Q_Key, XRCETH, RDETH

## 1. 总览

### 1.1 九种扩展头

| 扩展头 | 长度 | 字段 | 出现于（OpCode 示例） | RoCEv2 实践 |
|---|---|---|---|---|
| RDETH | 4 B | Reserved(8) + EE-Context(24) | 所有 RD 包（0x40–0x55） | RD 基本无实现 |
| DETH | 8 B | Q_Key(32) + Reserved(8) + SrcQP(24) | UD SEND（0x64/0x65）；RD 请求 | UD 常用 |
| XRCETH | 4 B | Reserved(8) + XRCSRQ(24) | XRC 请求（0xA0–0xAC、0xB3/0xB4、0xB6/0xB7） | 部分 RNIC 支持 |
| RETH | 16 B | VA(64) + R_Key(32) + DMA Length(32) | WRITE First/Only（含 with Immediate）、READ Request | 核心 |
| AtomicETH | 28 B | VA(64) + R_Key(32) + Swap/Add(64) + Compare(64) | CmpSwap、FetchAdd | 常用 |
| AETH | 4 B | Syndrome(8) + MSN(24) | Acknowledge、READ Response First/Last/Only、ATOMIC Acknowledge | 核心 |
| AtomicAckETH | 8 B | OrigRemDt(64) | ATOMIC Acknowledge | 常用 |
| ImmDt | 4 B | Immediate Data(32) | SEND / WRITE 的 Last/Only with Immediate | 常用 |
| IETH | 4 B | R_Key(32) | SEND Last/Only with Invalidate（RC、XRC） | 常用 |

〔Vol1 §5.2.4–§5.2.12, p.174–178；Figure 46, p.170〕。RoCEv2 CNP 在 BTH 之后有 16 字节保留区（Figure 349），它不是上述任何一种扩展头，见 [05-opcodes.md](05-opcodes.md)。

### 1.2 固定的排列次序

同一个包中出现多个扩展头时，次序固定为（Figure 46 与 Table 38 的列举顺序一致）：

```text
BTH → RDETH → DETH → XRCETH → RETH → AtomicETH → AETH → AtomicAckETH → ImmDt → IETH → Payload → pad → ICRC
```

例如 RC "RDMA WRITE Only with Immediate" 为 `BTH, RETH, ImmDt, PayLd`；XRC 同名操作为 `BTH, XRCETH, RETH, ImmDt, PayLd`；RD 为 `BTH, RDETH, DETH, RETH, ImmDt, PayLd`〔Vol1 Table 38, p.257–259〕。次序固定意味着每个头的偏移只取决于它前面有哪些头 —— 这是硬件用"存在位图 + 前缀和"一拍算出所有偏移的基础（§10）。

所有多字节字段都按大端（网络序）传输，位编号约定同 [03-bth.md](03-bth.md)（bit 31 为每个 32 bit 字的最高位）。

## 2. RETH —— RDMA 扩展传输头（16 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------------------------------------------------------+
 0-3   |                    Virtual Address [63:32]                    |
       +---------------------------------------------------------------+
 4-7   |                    Virtual Address [31:0]                     |
       +---------------------------------------------------------------+
 8-11  |                             R_Key                             |
       +---------------------------------------------------------------+
12-15  |                          DMA Length                           |
       +---------------------------------------------------------------+
```

| 偏移 | 字段 | 宽度 | 含义与规则 |
|---|---|---|---|
| 0–7 | VA | 64 | 远端缓冲区起始虚拟地址，**可以从任意字节边界开始**〔Vol1 §9.3.3.1, p.263〕 |
| 8–11 | R_Key | 32 | 远端密钥。响应方必须校验 R_Key、访问权限、虚拟地址，并做边界检查；违例时丢弃该包，可靠服务还要回 NAK〔Vol1 §9.3.3.2, p.263–264〕 |
| 12–15 | DMA Length | 32 | 整个 RDMA 操作的字节数，范围 0 – 2³¹（C9-9）〔Vol1 §9.3.3.3, p.264〕 |

出现于：RC/UC/RD/XRC 的 RDMA WRITE First、RDMA WRITE Only、RDMA WRITE Only with Immediate，以及 RC/RD/XRC 的 RDMA READ Request〔Vol1 Table 38〕。

要点：
- RETH **只出现在消息的第一个（或唯一的）包里**〔Vol1 §9.4.3, p.278〕。WRITE 的 Middle/Last 包不再携带地址，响应方必须把 VA、R_Key、剩余长度保存在 QP 上下文中，按包顺序累加。
- RDMA READ：请求方据 DMA Length 和 PMTU 算出响应包数 `n = max(1, ⌈DMALen / PMTU⌉)`，下一个请求的 PSN = READ PSN + n（Table 42）〔Vol1 §9.7.3.1, p.314〕。零长度 READ 得到一个零负载的 "READ Response Only"〔Vol1 §9.4.4, p.281〕。
- 长度校验：READ 请求的 DMA Length > 2³¹ 回 NAK-Invalid Request（C9-94）；WRITE 的 DMA Length 可选检查，响应方也可在消息末尾核对各包负载之和是否等于 DMA Length〔Vol1 §9.7.4.1.6, p.330〕。
- R_Key 违例（无效 R_Key、越界、权限不符、PD 不符）在 RC/XRC 中回 NAK-Remote Access Error（§9.7.5.2.4）；UC 没有 NAK，非 SRQ 的 UC QP 按 Responder Class D 静默丢弃〔Vol1 §9.9.3 Table 61, p.437；§9.9.3.1.4, p.443〕。R_Key 与内存保护的设计见 [内存管理](../03-verbs-and-resources/03-memory-management.md)。

## 3. AtomicETH —— 原子操作扩展传输头（28 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------------------------------------------------------+
 0-3   |                    Virtual Address [63:32]                    |
       +---------------------------------------------------------------+
 4-7   |                    Virtual Address [31:0]                     |
       +---------------------------------------------------------------+
 8-11  |                             R_Key                             |
       +---------------------------------------------------------------+
12-15  |                  Swap (or Add) Data [63:32]                   |
       +---------------------------------------------------------------+
16-19  |                   Swap (or Add) Data [31:0]                   |
       +---------------------------------------------------------------+
20-23  |                     Compare Data [63:32]                      |
       +---------------------------------------------------------------+
24-27  |                      Compare Data [31:0]                      |
       +---------------------------------------------------------------+
```

| 偏移 | 字段 | 宽度 | 含义 |
|---|---|---|---|
| 0–7 | VA | 64 | 目标地址，**必须 8 字节自然对齐**（低 3 位为 0），否则响应方回 NAK-Invalid Request〔Vol1 §9.4.5, p.283；p.290〕 |
| 8–11 | R_Key | 32 | 远端密钥；R_Key 范围必须完整覆盖这 8 个字节〔Vol1 p.290〕 |
| 12–19 | SwapDt / AddDt | 64 | CmpSwap：比较相等时写入的值；FetchAdd：加数〔Vol1 §9.3.4.3, p.265〕 |
| 20–27 | CmpDt | 64 | CmpSwap 的比较值〔Vol1 §9.3.4.4, p.265〕；FetchAdd 不使用 |

语义〔Vol1 §9.4.5, p.283–285〕：

| 操作 | OpCode（RC / RD / XRC） | 响应方动作 | 返回 |
|---|---|---|---|
| CmpSwap | 0x13 / 0x53 / 0xB3 | 读 64 bit 原值；若等于 CmpDt 则写入 SwapDt，否则不变 | 原值 |
| FetchAdd | 0x14 / 0x54 / 0xB4 | 读 64 bit 原值，加上 AddDt（无符号加法，进位丢弃）后写回 | 原值 |

- 原子操作只用于 RC、RD、XRC（UC/UD 不支持）〔Vol1 Table 39, p.268〕，不带 Immediate，强烈建议完全由硬件实现。
- 线上操作数一律大端；响应方按目标内存的字节序转换后执行，返回的原值再转回大端〔Vol1 §9.4.5, p.283–285〕。
- 原子请求与 RDMA READ 一起占用协商的"未完成 READ/ATOMIC 数"资源；重复的原子请求必须返回缓存的原结果而不能再执行一次，见 [响应方行为](../02-transport/07-responder.md)。

## 4. AETH —— 确认扩展传输头（4 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------+-----------------------------------------------+
 0-3   |   Syndrome    |         MSN (Message Sequence Number)         |
       +---------------+-----------------------------------------------+
```

Syndrome 字节内部：

```text
    7   6   5   4   3   2   1   0   (bit)
  +---+-------+-------------------+
  | 0 | Type  |       Value       |
  +---+-------+-------------------+
```

bit 7 保留为 0，bits 6:5 为类型，bits 4:0 的含义由类型决定（见下表）。

AETH 出现在 Acknowledge、RDMA READ Response First / Last / Only 和 ATOMIC Acknowledge 中；READ Response Middle **不带** AETH（C9-24、C9-117、C9-118）〔Vol1 §9.3.5, p.265；§9.4.4, p.282；§9.7.5.1.9, p.346〕。

### 4.1 Syndrome 编码（Table 46）

| bit 7 | bits 6:5 | bits 4:0 | 含义 | Syndrome 十六进制 |
|---|---|---|---|---|
| 0 | 00 | C CCCC | ACK，低 5 位为端到端信用（credit） | 0x00–0x1F（0x1F = 信用无效） |
| 0 | 01 | T TTTT | RNR NAK，低 5 位为 RNR 定时器 | 0x20–0x3F |
| 0 | 10 | X XXXX | 保留 | 0x40–0x5F |
| 0 | 11 | N NNNN | NAK，低 5 位为 NAK 码 | 0x60–0x64 有效，0x65–0x7F 保留 |

- bit 7 保留，必须为 0（C9-119.a1）；HCA 的 RC 响应方必须按此表编码（C9-119）〔Vol1 §9.7.5.2, p.348〕。
- NAK（011）同样携带 MSN〔Vol1 p.349〕。

### 4.2 NAK 码（Table 47）

| NAK 码（bits 4:0） | Syndrome | 含义 | BTH.PSN 的取值 |
|---|---|---|---|
| 0 0000 | 0x60 | PSN Sequence Error | 响应方的期望 PSN（C9-121） |
| 0 0001 | 0x61 | Invalid Request | 出错请求包的 PSN，即期望 PSN（C9-125） |
| 0 0010 | 0x62 | Remote Access Error | 造成访问错误的请求包 PSN（C9-124） |
| 0 0011 | 0x63 | Remote Operational Error | 检测到错误时正在执行的请求 PSN（C9-126） |
| 0 0100 | 0x64 | Invalid RD Request | （RD 专用） |
| 0 0101 – 1 1111 | 0x65–0x7F | 保留 | —— |

〔Vol1 §9.7.5.2.2–§9.7.5.2.7, p.349–352〕。请求方收到保留码的确认包视为畸形包，**静默丢弃**（C9-120），这可能导致请求方最终超时重传。各类 NAK 的产生条件与恢复行为见 [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md) 与 [错误处理](../02-transport/08-error-handling.md)。

### 4.3 RNR NAK 定时器编码（Table 48，完整 32 项）

| 码值 | 延迟（ms） | 码值 | 延迟（ms） | 码值 | 延迟（ms） | 码值 | 延迟（ms） |
|---|---|---|---|---|---|---|---|
| 00000 | 655.36 | 01000 | 0.16 | 10000 | 2.56 | 11000 | 40.96 |
| 00001 | 0.01 | 01001 | 0.24 | 10001 | 3.84 | 11001 | 61.44 |
| 00010 | 0.02 | 01010 | 0.32 | 10010 | 5.12 | 11010 | 81.92 |
| 00011 | 0.03 | 01011 | 0.48 | 10011 | 7.68 | 11011 | 122.88 |
| 00100 | 0.04 | 01100 | 0.64 | 10100 | 10.24 | 11100 | 163.84 |
| 00101 | 0.06 | 01101 | 0.96 | 10101 | 15.36 | 11101 | 245.76 |
| 00110 | 0.08 | 01110 | 1.28 | 10110 | 20.48 | 11110 | 327.68 |
| 00111 | 0.12 | 01111 | 1.92 | 10111 | 30.72 | 11111 | 491.52 |

〔Vol1 §9.7.5.2.8 Table 48, p.354–355〕

- 响应方在 RNR NAK 的定时器字段中填入合适的等待间隔（C9-129），HCA 因没有可用接收 WQE 而回 RNR NAK 时，该值取自 QP 的配置（Table 63 注 b）；请求方至少等待该时间后才能从同一 PSN 重发同一请求（C9-128、C9-130），RNR 重试计数为 3 bit，7 表示无限重试（C9-132）〔Vol1 p.353–354；p.452〕。
- 注意 **00000 表示最长的 655.36 ms**，不是 0。

### 4.4 端到端信用编码（Table 50，完整 32 项）

ACK 的低 5 位编码的是"相对 MSN 还能再发多少条消耗接收 WQE 的消息"，请求方的 **LSN（Limit Sequence Number）= MSN + 信用值**〔Vol1 §9.7.7.2.4, p.378–379〕：

| 码值 | 信用 | 码值 | 信用 | 码值 | 信用 | 码值 | 信用 |
|---|---|---|---|---|---|---|---|
| 00000 | 0 | 01000 | 16 | 10000 | 256 | 11000 | 4096 |
| 00001 | 1 | 01001 | 24 | 10001 | 384 | 11001 | 6144 |
| 00010 | 2 | 01010 | 32 | 10010 | 512 | 11010 | 8192 |
| 00011 | 3 | 01011 | 48 | 10011 | 768 | 11011 | 12288 |
| 00100 | 4 | 01100 | 64 | 10100 | 1024 | 11100 | 16384 |
| 00101 | 6 | 01101 | 96 | 10101 | 1536 | 11101 | 24576 |
| 00110 | 8 | 01110 | 128 | 10110 | 2048 | 11110 | 32768 |
| 00111 | 12 | 01111 | 192 | 10111 | 3072 | 11111 | **无效** |

- 响应方填入**不超过实际可用 WQE 数的最大可表示值**：规范的例子是"有 5 个信用时填 00100（即 4）"（C9-158.2.1）〔Vol1 §9.7.7.2.3, p.377〕。
- 信用只对 RC 有意义：RD 置 0（o9-72），XRC 响应的信用总是无效（Table 63 注 c），关联 SRQ 的 QP 以及不产生信用的响应方填 11111（o9-95.2.2、C9-154）〔Vol1 p.348、p.452、p.373〕。
- 只有 SEND 和 RDMA WRITE with Immediate 消耗接收 WQE、需要信用；READ、不带 Immediate 的 WRITE、ATOMIC 可以无视信用发送（C9-162）〔Vol1 §9.7.7.2.5, p.381〕。

> [!TIP]
> 「硬件实现要点」信用表与 RNR 表有同一个规律：设 `V(0)=0, V(1)=1`，对 `c ≥ 2` 有 `V(c) = (2 + (c & 1)) << ((c >> 1) − 1)`，即 2, 3, 4, 6, 8, 12, 16, 24 …。信用值 = `V(c)`（c = 0…30）；RNR 延迟 = `10 µs × V(c)`（c = 1…31，其中 `V(31) = 49152`），而 c = 0 为 `10 µs × 65536 = 655.36 ms`。两张表都可以用"移位 + 选择"实现，不必存 ROM（规律已用脚本对 Table 48/50 全部 64 项核对）。

编码函数参考（Python，已运行验证）：

```python
CREDIT = [0, 1, 2, 3, 4, 6, 8, 12, 16, 24, 32, 48, 64, 96, 128, 192,
          256, 384, 512, 768, 1024, 1536, 2048, 3072, 4096, 6144, 8192,
          12288, 16384, 24576, 32768]          # 码值 0..30；31 = 无效


def encode_credit(n_wqe: int) -> int:
    """返回 AETH Syndrome 低 5 位：不超过 n_wqe 的最大可表示信用。"""
    return max(code for code, value in enumerate(CREDIT) if value <= n_wqe)


assert encode_credit(5) == 0b00100            # 规范示例：5 个信用 → 00100
```

### 4.5 MSN（24 bit）

- 响应方**最近完成的请求消息**的序号，用于让请求方高效地批量完成 WQE〔Vol1 §9.3.5.2, p.266〕。
- 无论是 ACK、NAK 还是重复包的确认，RC 响应方都要填入当前 MSN（C9-149）〔Vol1 §9.7.7.1.1, p.370〕；它也是信用的基准（LSN = MSN + 信用）。
- MSN 的递增规则、与请求方 SSN 的对应关系见 [PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md)。

## 5. AtomicAckETH —— 原子确认扩展传输头（8 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------------------------------------------------------+
 0-3   |                 Original Remote Data [63:32]                  |
       +---------------------------------------------------------------+
 4-7   |                  Original Remote Data [31:0]                  |
       +---------------------------------------------------------------+
```

- OrigRemDt：原子操作执行**之前**目标地址中的 64 bit 原值〔Vol1 §9.3.5.4, p.266〕。
- 只出现在 ATOMIC Acknowledge（RC 0x12、RD 0x52、XRC 0xB2），排在 AETH 之后，包内无负载。
- 以大端传输，由目标内存字节序转换而来；请求方再按本地字节序写回 WR 指定的本地缓冲区〔Vol1 §9.4.5, p.284–285〕。

## 6. ImmDt —— 立即数（4 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------------------------------------------------------+
 0-3   |                        Immediate Data                         |
       +---------------------------------------------------------------+
```

- 32 bit 不透明数据，响应方把它放进接收端 CQE〔Vol1 §9.3.6, p.266〕。
- 只出现在 "with Immediate" 的 Last/Only 包中：RC 0x03/0x05/0x09/0x0B，UC 0x23/0x25/0x29/0x2B，UD 0x65，RD 与 XRC 中的同名操作。
- RDMA WRITE with Immediate 的数据写入远端内存，立即数则**消耗一个接收 WQE** 并产生接收完成，因此它需要信用（§4.4）。
- 〔业界实践: libibverbs〕 Verbs API 中 `imm_data` 声明为 `__be32`，应用按网络序填写与读取。

## 7. IETH —— 失效扩展传输头（4 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------------------------------------------------------+
 0-3   |                             R_Key                             |
       +---------------------------------------------------------------+
```

- 携带响应方需要失效（invalidate）的 R_Key，出现在 SEND Last / Only with Invalidate（RC 0x16/0x17、XRC 0xB6/0xB7）〔Vol1 §9.3.7, p.266–267〕。
- IETH 的 R_Key **不在传输层报文头校验中检查**，由通道接口在执行失效前校验（o9-5.2.2）；失效必须在常规报文头校验通过之后才执行（o9-5.2.1），且 ACK/NAK 不反映失效成功与否〔Vol1 §9.4.1.1, p.272–274〕。
- 失效操作与之后的操作之间没有顺序保证；有未完成 READ 时不应发送 SEND with Invalidate，必要时使用 Fence〔Vol1 §9.4.1.1.1, p.273〕。内存失效的语义见 [内存管理](../03-verbs-and-resources/03-memory-management.md)。

## 8. DETH —— 数据报扩展传输头（8 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------------------------------------------------------+
 0-3   |                             Q_Key                             |
       +---------------+-----------------------------------------------+
 4-7   |   Reserved    |                   Source QP                   |
       +---------------+-----------------------------------------------+
```

| 偏移 | 字段 | 宽度 | 规则 |
|---|---|---|---|
| 0–3 | Q_Key | 32 | 授权访问目的接收队列；响应方与目的 QP 的 Q_Key 比较〔Vol1 §9.3.2.1, p.263〕 |
| 4 | Reserved | 8 | 发 0 收忽略（C9-8） |
| 5–7 | SrcQP | 24 | 源 QP 号，响应方回包时作为目的 QP〔Vol1 §9.3.2.3, p.263〕；UD 接收完成中上报给应用 |

Q_Key 规则：
- **最高位为 1 的 Q_Key 是"受控 Q_Key"**（如 GSI 的 Q_Key），普通用户不能随意使用〔Vol1 §3.5.3, p.120〕。发送时通道接口检查 WR 中的 Q_Key：**若其最高位为 1，报文中填 QP 上下文里的 Q_Key；否则填 WR 中的 Q_Key**（C10-15）〔Vol1 §10.2.5, p.471〕。这样只有被特权软件配置过的 QP 才能发出受控 Q_Key。
- 接收：发往 QP1 的报文只要 Q_Key = 0x80010000 即有效（C9-49）；其它 UD QP 的 Q_Key 不匹配则静默丢弃（C9-50、C10-14）〔Vol1 §9.6.1.5.1, p.301；p.470〕。

出现于 UD SEND Only（0x64）、UD SEND Only with Immediate（0x65）以及 RD 的所有请求包。RoCEv2 UD 的完成信息额外包含源 MAC 和"IPv4 / IPv6 / RoCE"标志，接收缓冲前 40 字节存放 IP 头（IPv4 头位于偏移 20）〔Vol1 §A17.4.5, p.1939〕，详见 [UC/UD/多播](../02-transport/04-unreliable-services.md)。

## 9. XRCETH 与 RDETH

### 9.1 XRCETH（4 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------+-----------------------------------------------+
 0-3   |   Reserved    |                    XRCSRQ                     |
       +---------------+-----------------------------------------------+
```

- XRCSRQ 指定响应方使用哪个 XRC 共享接收队列〔Vol1 §9.3.8, p.267〕；Reserved 发 0 收忽略。
- 只出现在 XRC **请求**包中（0xA0–0xAC、0xB3、0xB4、0xB6、0xB7）；XRC 的响应包（0xAD–0xB2）不带 XRCETH。
- 接收检查：XRC SRQ 必须与 XRC 目标 QP 属于同一 XRC 域（§9.6.1.3, p.300）。XRC 在 RoCE 上的支持情况厂商相关，服务模型见 [服务类型](../02-transport/01-service-types.md)。

### 9.2 RDETH（4 B）

```text
 字节   31           24 23           16 15            8 7             0
       +---------------+-----------------------------------------------+
 0-3   |   Reserved    |                  EE-Context                   |
       +---------------+-----------------------------------------------+
```

- 标识可靠数据报（RD）使用的端到端上下文（EE Context）〔Vol1 §9.3.1, p.262〕，出现在所有 RD 包（0x40–0x55）的最前面。
- 〔业界实践〕 RD 服务在商用 RNIC 与 Linux libibverbs 中基本没有实现（libibverbs 没有 RD 类型的 QP），RoCEv2 实现可以不支持；收到 RD OpCode 而本端不支持 RD 时静默丢弃（C9-47.2.1）〔Vol1 §9.6.1.4.1, p.300〕。

## 10. 「硬件实现要点」

1. **偏移前缀和**：OpCode 译码得到 9 个"存在位"，按 §1.2 的固定次序做前缀和即可得到各头偏移与负载起点（以 BTH 起点为 0）：

   ```systemverilog
   // has_* 由 OpCode 查表得到（见 05-opcodes.md）
   always_comb begin
       off           = 7'd12;                         // BTH 之后
       rdeth_off     = off;  off += has_rdeth     ? 4  : 0;
       deth_off      = off;  off += has_deth      ? 8  : 0;
       xrceth_off    = off;  off += has_xrceth    ? 4  : 0;
       reth_off      = off;  off += has_reth      ? 16 : 0;
       atomiceth_off = off;  off += has_atomiceth ? 28 : 0;
       aeth_off      = off;  off += has_aeth      ? 4  : 0;
       atomack_off   = off;  off += has_atomack   ? 8  : 0;
       immdt_off     = off;  off += has_immdt     ? 4  : 0;
       ieth_off      = off;  off += has_ieth      ? 4  : 0;
       payload_off   = off;                           // 最大 12 + 40 = 52（RD 原子请求）
   end
   ```

2. **非对齐的 64 bit 字段**：RETH 的 VA 在 IPv4 无 VLAN 时位于帧偏移 54，在 8 字节总线通道上并不对齐；按字节抽取再拼接，不要假设自然对齐。
3. **多包 WRITE 的上下文**：First 包解析出的 VA、R_Key 与剩余长度写入 QP 上下文，后续 Middle/Last 包据此生成 DMA 地址并做边界检查（地址翻译与保护检查见 [内存管理](../03-verbs-and-resources/03-memory-management.md)）。
4. **AETH 解析**：`syndrome[6:5]` 选择类型；NAK 码 > 4 或类型 = 10b 属于畸形包，静默丢弃并计数（C9-120）；信用与 RNR 定时器按 §4.4 的移位规律实时换算。
5. **原子操作**：先检查 `VA[2:0] == 0`（否则 NAK-Invalid Request），再做 R_Key 检查；执行单元必须保证读-改-写的原子性，并为重复请求缓存结果（[可靠性引擎](../06-hardware-implementation/04-reliability-engine.md)）。
6. **IETH**：只把 R_Key 交给失效逻辑，失效动作必须在 ICRC 与报文头校验都通过之后才执行。

## 11. 要点回顾

- 九种扩展头按固定次序排列：RDETH → DETH → XRCETH → RETH → AtomicETH → AETH → AtomicAckETH → ImmDt → IETH。
- RETH（16 B）只在 WRITE First/Only 和 READ Request 中出现，VA 可任意对齐，DMA Length 最大 2³¹。
- AtomicETH（28 B）的 VA 必须 8 字节对齐；CmpSwap 用 Swap + Compare，FetchAdd 只用 Add；结果原值由 AtomicAckETH（8 B）带回。
- AETH = Syndrome(8) + MSN(24)：00 = ACK（信用）、01 = RNR NAK（定时器）、10 = 保留、11 = NAK（0–4 有效）。
- RNR 定时器 00000 = 655.36 ms（最长），00001 = 0.01 ms；信用 11111 表示无效，有 5 个 WQE 时编码为 00100（4）。
- ImmDt 消耗接收 WQE；IETH 的 R_Key 不在传输层校验；DETH 的 Q_Key 最高位为 1 时发送端改用 QP 上下文中的 Q_Key（C10-15）。
- RD/RDETH 在 RoCE 实践中基本不用；XRCETH 只在 XRC 请求包中出现。

## 自测

<details><summary>Q1：AETH 的 Syndrome 为 0x2E，代表什么？请求方该怎么做？</summary>

答：0x2E = 0_01_01110b，类型 01 为 RNR NAK，定时器码 01110 = 1.28 ms。请求方至少等待 1.28 ms 后，从该 NAK 中 PSN 指示的包开始重发同一请求，并递减 RNR 重试计数（计数为 7 时表示无限重试）。
</details>

<details><summary>Q2：一个 ACK 的 AETH 为 0x0A000105，请求方算出的 LSN 是多少？</summary>

答：Syndrome = 0x0A = 0_00_01010b，类型 ACK，信用码 01010 = 32；MSN = 0x000105。LSN = 0x105 + 32 = 0x125，请求方可以发送 SSN ≤ 0x125 的消耗 WQE 的消息。
</details>

<details><summary>Q3：UD 应用在 WR 中指定 Q_Key = 0x80010000 发给远端 QP1，报文中实际的 Q_Key 是什么？</summary>

答：0x80010000 的最高位为 1，是受控 Q_Key，按 C10-15 报文中填写的是本端 QP 上下文中的 Q_Key，而不是 WR 中的值。只有特权软件把该 QP 的 Q_Key 配置为 0x80010000（如 GSI/CM 使用的 QP）时，报文才会携带它。
</details>

<details><summary>Q4：RC 的 RDMA WRITE Only with Immediate 包中，BTH 之后各头的偏移是多少？</summary>

答：RETH 在 BTH 起点后 12 字节处（12–27），ImmDt 在 28–31，负载从 32 开始。换算到 IPv4 无 VLAN 帧：BTH 42，RETH 54，ImmDt 70，负载 74。
</details>

<details><summary>Q5：为什么 READ Response Middle 不带 AETH？</summary>

答：AETH 用于确认和携带 MSN/信用。READ 响应的 First（或 Only）包触发对之前未确认请求的隐式确认，Last（或 Only）包确认 READ 本身（§9.7.5.1.9），中间包只是数据，不需要再携带确认信息，因此规范规定 Middle 包不得包含 AETH（C9-24）。
</details>

## 参考

- 规范：Vol1 §3.5.3（p.120）；§5.2.4–§5.2.12 Table 7–15（p.174–178）；Figure 46（p.170）；§9.3.1–§9.3.8（p.262–267）；Table 38（p.257–259）、Table 39（p.268）；§9.4.1.1（p.272–274）；§9.4.3–§9.4.5（p.275–290）；§9.6.1.3–§9.6.1.5（p.300–302）；§9.7.3.1 Table 42（p.313–314）；§9.7.4.1.6（p.330）；§9.7.5.1.9（p.346–348）；§9.7.5.2 Table 46–48（p.348–355）；§9.7.7.1–§9.7.7.2 Table 50（p.370–381）；§9.10 Table 63（p.450–452）；§10.2.5（p.470–471）；§A17.4.5（p.1939）。
- 业界实践：libibverbs（`struct ibv_send_wr` / `ibv_wc` 的 `imm_data` 为 `__be32`，无 RD QP 类型）；Linux 内核 `drivers/infiniband/sw/rxe/rxe_hdr.h`。
- 相关文件：[03-bth.md](03-bth.md)、[05-opcodes.md](05-opcodes.md)、[操作与报文序列](../02-transport/02-operations.md)、[PSN 与 ACK/NAK](../02-transport/05-psn-and-ack-nak.md)、[请求方行为](../02-transport/06-requester.md)。
