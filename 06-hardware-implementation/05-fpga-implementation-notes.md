# FPGA 实现要点（FPGA Implementation Notes）

> **摘要**：面向在 FPGA 上实现 RoCEv2 RNIC 的工程师，本文给出：各速率档位的数据通路位宽/时钟与硬核 IP 选型要点；512 bit 并行 CRC32（ICRC）的矩阵推导、XOR 树规模与时序、"64 bit 全 1 前缀"与屏蔽字段的在线替换、尾拍字节掩码处理，并附经 iverilog 仿真验证的参考 RTL；变长头部的对齐与拼接；上下文存储的放置、缓存一致性、跨时钟域与时序收敛；资源受限器件上的精简方案及其合规风险；以及公开的 FPGA RDMA 参考实现对比。
>
> **规范出处**：Vol1 §A17.3.3（CA17-22, p.1935–1936）；§7.8.1、§7.8.4（p.218–231）；§17.2（p.1188–1203）；§A17.7（p.1944–1945）；§9.7.6.1.3（p.362–365）
>
> **前置阅读**：[RNIC 总体架构](01-rnic-architecture.md)、[发送数据通路](02-tx-datapath.md)、[接收数据通路](03-rx-datapath.md)、[ICRC 计算](../01-packet-format/06-icrc.md)
>
> **关键词**：AXI4-Stream, tkeep, 并行 CRC, GF(2) 矩阵, XOR 树, LUT6, 0xDEBB20E3, 尾拍回卷, 桶形移位器, BRAM/URAM/HBM, CDC, 精简 RNIC, ERNIC, RecoNIC, Coyote, StRoM

## 1. 速率档位、数据通路与硬核 IP

### 1.1 速率档位

| 速率 | 常见 MAC 用户侧接口 | 内部数据通路建议 | 64 B 帧包率 | 说明 |
|---|---|---|---:|---|
| 10G | 64 bit @ 156.25 MHz | 64 bit 同频，或 128 bit 降频 | 14.88 Mpps | 资源最省，中端器件即可 |
| 25G | 64 bit @ 390.625 MHz | 128 bit @ ≥ 200 MHz（经异步 FIFO 跨域） | 37.20 Mpps | 390 MHz 下做复杂逻辑困难，宜尽早加宽降频 |
| 100G | 512 bit @ 322.265625 MHz（如 AMD UltraScale+ CMAC〔外部: AMD PG203，以文档为准〕） | 512 bit @ 322 MHz；250 MHz 只适合大包为主的设计 | 148.81 Mpps | 100G 最坏拍对齐需要约 281 M 拍/秒（推导见 [发送数据通路](02-tx-datapath.md)） |
| 200G | 依 IP 而定 | 1024 bit @ ≥ 250 MHz 或 512 bit @ ≥ 450 MHz | 297.62 Mpps | — |
| 400G | 分段（segmented）接口，一拍可含多个包（以所用 MAC IP 文档为准） | 1024 bit 以上并支持每拍多包 | 595.24 Mpps | 单包每拍接口无法满足最小帧线速 |

> [!TIP]
> RoCE 的数据包大多是 1–4 KB 的大包，但 ACK（IPv4 总线上 62 B）、CNP、小消息会形成小包风暴。设计目标可以分两档：**数据面按 PMTU 大包线速**，**小包按"不死锁、不丢合法包、可降速"**保证正确性，并用 PFC/拥塞控制把压力推回对端。

### 1.2 以太网 MAC/PCS 硬核

- 首选器件自带的 MAC+PCS+FEC 硬核（如 AMD UltraScale+ 的 CMAC、Versal 的 MRMAC/DCMAC，Intel/Altera 的 E-tile/F-tile 以太网硬核；具体能力以官方文档为准）。软核 MAC 在 100G 以上代价很高。
- 选型核对项：
  1. **PFC**：能否按优先级发送与解析 IEEE 802.1Qbb PFC 帧、能否按优先级暂停发送（RoCEv2 无损网络的基础，§A17.9.1）；
  2. **RS-FEC**：100GBASE-CR4/SR4 等需要 RS-FEC〔外部: IEEE 802.3 Clause 91〕，会额外增加数十到上百 ns 时延（以器件数据为准）；
  3. **FCS 处理**：接收是否剥离 FCS、坏 FCS 如何在 tuser 上标示（帧尾才知道，需要与存储转发配合）；
  4. **发送欠载**：一帧开始后是否允许数据中断（多数不允许），决定发送侧是否必须逐包存储转发。

### 1.3 PCIe 硬核与 DMA

- 选项一：器件 PCIe 硬核 + 厂商 DMA IP（如 AMD XDMA/QDMA；RecoNIC 与 fpga-network-stack 的 RDMA 栈都基于 QDMA〔外部: 各项目 README〕）。上手快，但要核对：能否为每个 TLP 控制 Relaxed Ordering 等属性、未完成读请求（Tag）数量、读完成是否已按序重排、是否支持非对齐地址。
- 选项二：直接使用 PCIe 硬核的 TLP 级接口自研 DMA 引擎（如开源 NIC Corundum 的做法〔外部: Corundum 项目，以项目文档为准〕）。对 Tag、排序属性、读完成重排有完全控制，适合需要精细控制 CQE 可见性顺序的 RNIC。
- RNIC 对主机接口的特殊需求：可映射到用户态的 4 KB 对齐 Doorbell 页（每进程一页）、足够多的 MSI-X 向量（每 EQ 一个）、高并发 DMA 读、可选 SR-IOV/ATS。

### 1.4 时钟域规划

| 时钟域 | 典型频率 | 包含逻辑 |
|---|---|---|
| MAC 收发 | 322.265625 MHz（100G） | 解析、ICRC、包缓冲写入侧、PFC 接口 |
| 传输核心 | 250～350 MHz | 调度、QPC 读改写、可靠性引擎、分段/头生成 |
| PCIe/DMA | 250 MHz（常见） | DMA 读写、Doorbell、MSI-X |
| 片外存储 | 依 DDR4/HBM 控制器 | 上下文后备存储 |

跨域一律在"包"或"命令"边界用异步 FIFO（格雷码指针）完成；单比特控制用两级同步器；多位计数器（如定时器的全局 tick）只在一个域内维护，向其它域传递时用握手或格雷码（**设计建议**）。

## 2. 并行 CRC32（ICRC）

### 2.1 矩阵法推导

CRC 寄存器的更新在 GF(2) 上是线性的。设 32 bit 状态为 s，一拍输入 W 字节数据 d（8W bit），不做首尾取反的寄存器更新满足：

```text
s_next = A_W · s  ⊕  B_W · d          （所有运算为 GF(2)，⊕ 即 XOR）
A_W : 32×32 矩阵，等于"输入 W 个零字节"对状态的作用
B_W : 32×8W 矩阵，等于"状态为 0 时"数据对结果的作用
```

求矩阵不必手推多项式：用一个逐位的参考模型分别输入单位向量即可得到每一列——状态第 j 位置 1、数据全 0，得到 A_W 的第 j 列；状态为 0、数据第 i 位置 1，得到 B_W 的第 i 列。约定：AXI4-Stream 的第 k 字节通道 `tdata[8k+7:8k]` 是线上第 k 个字节，字节内最低位先进入 CRC（与以太网 FCS 一致，§7.8.1），因此数据位序号 i = 8k + b 与 `tdata` 位号一一对应，**寄存器保持"反射"形式（与 zlib 相同）就不需要任何位翻转**。下面的生成器输出每个状态位的 XOR 方程（已运行验证）：

```python
P = 0xEDB88320                      # 0x04C11DB7 的反射形式

def crc_upd(reg, data):             # 逐位参考模型：无首尾取反（等同 Linux crc32_le）
    for b in data:
        reg ^= b
        for _ in range(8):
            reg = (reg >> 1) ^ (P if reg & 1 else 0)
    return reg

def gen_verilog_xor(W):
    """生成 W 字节并行 CRC 的方程：n = A*c ^ B*d（c 为 32 bit 状态，d 为 8W bit 数据）"""
    A = [crc_upd(1 << j, bytes(W)) for j in range(32)]
    B = []
    for i in range(8 * W):
        d = bytearray(W)
        d[i // 8] = 1 << (i % 8)
        B.append(crc_upd(0, bytes(d)))
    out = []
    for r in range(32):
        terms = ["c[%d]" % j for j in range(32) if (A[j] >> r) & 1]
        terms += ["d[%d]" % i for i in range(8 * W) if (B[i] >> r) & 1]
        out.append("assign n[%d] = %s;" % (r, " ^ ".join(terms)))
    return "\n".join(out)
# gen_verilog_xor(1) 的第一行为: assign n[0] = c[2] ^ c[8] ^ d[2];
```

也可以不生成方程，而在 RTL 里写一个"按字节、按位循环"的函数，由综合工具常量折叠成 XOR 网络（见 2.7 的参考 RTL）；两种写法结果等价。

### 2.2 XOR 树规模与时序（脚本统计结果）

| 总线宽度 | 数据部分扇入 最大/平均 | 状态部分扇入最大 | 单周期总扇入最大 | 2 输入 XOR 级数 | LUT6 级数（单周期） | 状态环路 LUT6 级数 | 数据部分 LUT6（不共享估算） |
|---:|---|---:|---:|---:|---:|---:|---:|
| 64 bit | 34 / 29.3 | 19 | 52 | 6 | 3 | 2 | ≈ 192 |
| 128 bit | 69 / 62.3 | 20 | 89 | 7 | 3 | 2 | ≈ 404 |
| 256 bit | 138 / 124.2 | 21 | 157 | 8 | 3 | 2 | ≈ 804 |
| 512 bit | 271 / 249.7 | 21 | 288 | 9 | 4 | 2 | ≈ 1601 |
| 1024 bit | 533 / 507.5 | 21 | 554 | 10 | 4 | 2 | ≈ 3255 |

解读与建议（**设计建议**）：

- 512 bit 单周期实现每个输出位最多 288 个输入，需要 4 级 LUT6 外加大量布线，在 322 MHz（3.1 ns）下余量很小。
- 关键在于：**只有状态部分 A_W·s 处在反馈环里**，其扇入最多 21（再加 1 个数据部分结果）只需 2 级 LUT6；数据部分 B_W·d 没有反馈，可以任意加流水级。因此结构应为：数据部分 2～3 级流水 → 与状态环路相加。
- 表中 LUT6 数是"每个输出位独立建树"的上界估算；综合工具会做公共子表达式共享，实际值通常更低（以实现报告为准）。

### 2.3 "64 bit 全 1 前缀"的处理

CA17-22 要求先输入 64 bit 全 1，再从 IP 头首字节开始计算〔Vol1 §A17.3.3, p.1935〕。三个经脚本验证的事实：

1. 初值 0xFFFFFFFF 输入 8 个 0xFF 后，寄存器（反射形式、未取反）= **0xDEBB20E3**——这正是 Linux rxe 用作 IP 头起始种子的常数〔业界实践: Linux rxe `rxe_icrc.c`〕；
2. 初值 0xFFFFFFFF 输入 4 个 0xFF 后寄存器恰为 0；因此"初值 0 + 4 个 0xFF"与"初值 0xFFFFFFFF + 8 个 0xFF"等价；
3. 寄存器为 0 时输入任意个 0x00 字节，寄存器仍为 0。

由此得到一个**不需要移位器的起始处理技巧**：寄存器从 0 开始，对整帧（从 DMAC 起）逐字节替换——IP 头之前 4 个字节替换为 0xFF，更前面的字节替换为 0x00——然后直接按整拍计算。不论 IP 头在第 14、18 还是 22 字节（0/1/2 层 VLAN），都无需把流对齐到 IP 头。

### 2.4 屏蔽字段的在线替换

各封装下需替换为 0xFF 的字节位置（从 DMAC 起算；IPv6 首字节只替换低 4 位）：

| 封装 | 前缀 0xFF | IPv4 TOS / TTL / 校验和 或 IPv6 TC+Flow Label / Hop Limit | UDP 校验和 | BTH 第 4 字节 | 所在拍（64 B） |
|---|---|---|---|---|---|
| IPv4，无 VLAN | 10–13 | 15 / 22 / 24–25 | 40–41 | 46 | 均在第 0 拍 |
| IPv4，1 层 VLAN | 14–17 | 19 / 26 / 28–29 | 44–45 | 50 | 均在第 0 拍 |
| IPv4，2 层 VLAN | 18–21 | 23 / 30 / 32–33 | 48–49 | 54 | 均在第 0 拍 |
| IPv6，无 VLAN | 10–13 | 14(低 4 位)、15–17 / 21 | 60–61 | 66 | BTH 字节在第 1 拍 |
| IPv6，1 层 VLAN | 14–17 | 18(低 4 位)、19–21 / 25 | 64–65 | 70 | UDP 校验和起在第 1 拍 |
| IPv6，2 层 VLAN | 18–21 | 22(低 4 位)、23–25 / 29 | 68–69 | 74 | UDP 校验和起在第 1 拍 |

**硬件实现要点**：EtherType 与 VLAN 层数在第 0 拍内即可解出，替换掩码只涉及前两拍，可以用"6 种封装 × 2 拍"的小型掩码 ROM 或每字节通道的比较器生成。替换只作用于送入 CRC 的副本，**不改变实际发送/接收的数据**。

### 2.5 尾拍（tkeep）处理

最后一拍只有 k（1～W）个有效字节。可选方案：

| 方案 | 做法 | 代价 |
|---|---|---|
| A：每种长度一套矩阵 | 为 k = 1…W 各生成一套 B_k、A_k，按 k 选择 | 64 套 512 bit 级 XOR 网络，面积大 |
| B：补零后回卷（推荐） | 无效字节填 0 按整拍计算，再把多算的 W−k 个零字节"倒回去" | 6 级（log2 64）可选的 32×32 小矩阵，每级扇入 ≤ 22，可流水 |
| C：逐字节串行处理尾部 | 最后一拍逐字节/逐 8 字节计算 | 小包时吞吐下降，100G 下不可接受 |
| D：头部补零右对齐 | 已知包长时在包头前补零，使包尾落在拍边界 | 需要包长先验与整流移位器 |

方案 B 的原理：寄存器处理零字节是可逆的线性变换 Z，填充 W−k 个零字节后 `S_full = Z^(W−k) · S_true`，所以 `S_true = Z^−(W−k) · S_full`。单个零比特的逆变换非常简单（因 0xEDB88320 的最高位为 1）：

```text
若 c[31] = 1：c = ((c ⊕ 0xEDB88320) << 1) | 1
否则：        c = c << 1
```

把 W−k 按二进制分解为 1、2、4、8、16、32 字节的组合，6 级各自是一个常量矩阵（实测每级最大扇入依次为 9、14、22、21、20、20），每级 2 个 LUT6 层，可插流水寄存器；回卷结果不进入反馈环，因此不影响吞吐。

### 2.6 接收侧：比较或余式检查

- 方式一：计算到 ICRC 之前，与收到的 4 字节比较。
- 方式二（余式）：把收到的 ICRC 也送入计算，正确报文的寄存器（未取反）恒为 **0xDEBB20E3**，即参考 RTL 的输出 `m_icrc` 恒为 **0x2144DF1C**（已用 200 帧随机报文仿真验证）。
- 无论哪种方式，`tkeep` 必须截止到"IP 长度所指的报文末尾"，**不得包含以太网短帧补齐字节**。

### 2.7 参考 RTL（已验证）

下面的模块用 iverilog（`-g2012`）仿真验证：300 帧随机 RoCEv2 报文（IPv4/IPv6 × 0/1/2 层 VLAN、负载 0～4096 B、拍间随机空闲、无效字节通道填 X），输出与 Python 参考实现（按 CA17-22 构造屏蔽字节流后调用 `zlib.crc32`）逐帧一致；参考实现本身已用 §7.8.4 的 Table 28/33 样例（0x9625B75A、0xB67D1ED1）和 Scapy `scapy.contrib.roce` 的 ICRC 交叉验证。

```verilog
// roce_icrc.v : RoCEv2 ICRC generator/checker core (documentation reference, verified with iverilog)
// Input: raw Ethernet frame bytes (starting at DMAC), without FCS and without ICRC.
//        Byte 0 of a beat = s_data[7:0] (AXI4-Stream little-endian byte lanes).
// Side-band (valid during the whole frame): l3off = byte offset of IP header (14/18/22),
//        ipv6 = 1 for IPv6. All masked fields lie in the first 2 beats when W >= 64.
// Output: one m_valid pulse per frame, m_icrc = ICRC value (zlib-compatible CRC-32);
//        on the wire the ICRC bytes are m_icrc[7:0], m_icrc[15:8], m_icrc[23:16], m_icrc[31:24].
module roce_icrc #(
  parameter integer W    = 64,              // bytes per beat (64 -> 512-bit)
  parameter integer LOGW = 6
)(
  input  wire           clk,
  input  wire           rst,
  input  wire           s_valid,
  input  wire [8*W-1:0] s_data,
  input  wire [W-1:0]   s_keep,             // contiguous from bit 0; all ones except on last beat
  input  wire           s_last,
  input  wire [7:0]     l3off,
  input  wire           ipv6,
  output reg            m_valid,
  output reg  [31:0]    m_icrc
);
  localparam [31:0] P = 32'hEDB88320;       // reflected 0x04C11DB7

  // CRC register update over W bytes, no pre/post inversion (== Linux crc32_le)
  function automatic [31:0] upd(input [31:0] c_in, input [8*W-1:0] d);
    integer i, b; reg [31:0] c;
    begin
      c = c_in;
      for (i = 0; i < W; i = i + 1) begin
        c = c ^ {24'd0, d[8*i +: 8]};
        for (b = 0; b < 8; b = b + 1) c = (c >> 1) ^ (P & {32{c[0]}});
      end
      upd = c;
    end
  endfunction

  // inverse of "process nbytes zero bytes"
  function automatic [31:0] rewind(input [31:0] c_in, input integer nbytes);
    integer i; reg [31:0] c;
    begin
      c = c_in;
      for (i = 0; i < 8 * nbytes; i = i + 1)
        c = c[31] ? (((c ^ P) << 1) | 32'd1) : (c << 1);
      rewind = c;
    end
  endfunction

  // ---------- stage 0: per-byte substitution (CA17-22 masking + 64-bit '1' prefix trick) ----------
  reg  [15:0]    beat_idx;                  // beat number inside current frame
  reg  [8*W-1:0] sub;
  integer j, pos, l4, bth;
  always @* begin
    l4  = l3off + (ipv6 ? 40 : 20);
    bth = l4 + 8;
    for (j = 0; j < W; j = j + 1) begin
      pos = beat_idx * W + j;
      sub[8*j +: 8] = s_keep[j] ? s_data[8*j +: 8] : 8'h00;       // invalid tail bytes -> 0
      if (pos < l3off - 4)                         sub[8*j +: 8] = 8'h00; // before prefix: zero (state is 0)
      else if (pos < l3off)                        sub[8*j +: 8] = 8'hFF; // 4x0xFF from state 0 == FFFFFFFF + 8x0xFF
      else if (!ipv6 && (pos == l3off + 1 || pos == l3off + 8 ||
                         pos == l3off + 10 || pos == l3off + 11))
                                                   sub[8*j +: 8] = 8'hFF; // IPv4 TOS, TTL, Header Checksum
      else if (ipv6 && pos == l3off)               sub[8*j +: 8] = s_data[8*j +: 8] | 8'h0F; // TC[7:4]
      else if (ipv6 && (pos == l3off + 1 || pos == l3off + 2 ||
                        pos == l3off + 3 || pos == l3off + 7))
                                                   sub[8*j +: 8] = 8'hFF; // TC[3:0]+FlowLabel, HopLimit
      else if (pos == l4 + 6 || pos == l4 + 7)     sub[8*j +: 8] = 8'hFF; // UDP checksum
      else if (pos == bth + 4)                     sub[8*j +: 8] = 8'hFF; // BTH F/B/Resv6
    end
  end

  // ---------- stage 1: data part F(0, D) (no feedback -> may be pipelined further) ----------
  reg [31:0]   s1_dpart;
  reg          s1_valid, s1_last, s1_sop;
  reg [LOGW:0] s1_npad;                     // number of zero bytes padded in last beat
  reg          sop;
  integer q; reg [LOGW:0] cnt;
  always @* begin
    cnt = 0;
    for (q = 0; q < W; q = q + 1) cnt = cnt + s_keep[q];
  end
  always @(posedge clk) begin
    if (rst) begin
      s1_valid <= 1'b0; sop <= 1'b1; beat_idx <= 16'd0;
    end else begin
      s1_valid <= s_valid;
      if (s_valid) begin
        s1_dpart <= upd(32'd0, sub);
        s1_last  <= s_last;
        s1_sop   <= sop;
        s1_npad  <= W - cnt;
        sop      <= s_last;
        beat_idx <= s_last ? 16'd0 : beat_idx + 16'd1;
      end
    end
  end

  // ---------- stage 2: state loop S <= (sop ? 0 : A*S) ^ dpart  (only 32+1 inputs per bit) ----------
  reg  [31:0]  st, s2_state;
  reg          s2_valid;
  reg  [LOGW:0] s2_npad;
  wire [31:0]  st_next = (s1_sop ? 32'd0 : upd(st, {8*W{1'b0}})) ^ s1_dpart;
  always @(posedge clk) begin
    if (rst) begin
      s2_valid <= 1'b0; st <= 32'd0;
    end else begin
      s2_valid <= s1_valid & s1_last;
      if (s1_valid) begin
        st <= st_next;
        if (s1_last) begin s2_state <= st_next; s2_npad <= s1_npad; end
      end
    end
  end

  // ---------- stage 3: tail rewind by npad bytes, binary decomposition (pipeline it for 300+ MHz) ----------
  reg [31:0] tail [0:LOGW];
  integer k;
  always @* begin
    tail[0] = s2_state;
    for (k = 0; k < LOGW; k = k + 1)
      tail[k+1] = s2_npad[k] ? rewind(tail[k], 1 << k) : tail[k];
  end
  always @(posedge clk) begin
    if (rst) m_valid <= 1'b0;
    else begin
      m_valid <= s2_valid;
      if (s2_valid) m_icrc <= ~tail[LOGW];
    end
  end
endmodule
```

使用说明与可改进处：

- 这是**功能参考**而非时序优化版本：stage 0 的逐字节比较器与 stage 1 的 512 bit 数据部分同在一拍，stage 3 的 6 级回卷是组合逻辑。上 322 MHz 时，应把替换掩码改为查表、把数据部分拆成 2～3 级流水、把回卷每 1～2 级插一级寄存器。
- 发送侧使用时，`s_keep` 截止到最后一个负载/填充字节，`m_icrc` 按小端追加到包尾（`m_icrc[7:0]` 先发）；接收侧可把 ICRC 也送入，检查 `m_icrc == 32'h2144DF1C`。
- 需要 `l3off` 与 `ipv6` 在帧的第 0 拍就有效；接收侧可在第 0 拍组合解析 EtherType/VLAN 得到，或把数据流延迟一拍。

## 3. 头部处理：变长头部的对齐与拼接

### 3.1 偏移为什么不固定

- 以太网头 14 B + 每层 VLAN 4 B，IP 头 20 B 或 40 B，UDP 8 B，BTH 12 B，扩展头 0～28 B：BTH 起点在 42～70 字节之间（见 [接收数据通路](03-rx-datapath.md) §2.2），负载起点 = BTH + 12 + 扩展头长度，例如 IPv4 无 VLAN 的 WRITE First 负载从第 70 字节开始。
- 除以太网头外，所有头长都是 4 的倍数，所以**头部总长恒满足 ≡ 2 (mod 4)**；但负载来自主机内存，其起始地址按字节任意对齐，因此仍然需要字节粒度的移位。

### 3.2 接收侧

1. 头部抽取：把前两拍收齐，按"IP 版本 × VLAN 层数"6 种变体做固定偏移抽取（比通用移位器省）；
2. 负载对齐：负载在流中的起点（帧内偏移 mod 64）与目标主机地址的对齐（VA mod 64）一般不同，写 DMA 前需要一个**字节旋转器 + 一拍暂存寄存器**，把负载重排成"按目标地址对齐的拍"，首末拍配合字节使能；若所用 DMA IP 支持非对齐 AXI-MM 写，可省掉这一步（以 IP 文档为准）。

### 3.3 发送侧

- 头部生成器先输出头部拍（最后一拍未满），负载流从 DMA 读回时按主机地址对齐；拼接器把负载旋转 `(头长 − 负载源地址) mod 64` 字节，与头部最后一拍合并，此后每拍"上一拍余下的字节 + 本拍字节"组成输出。
- ICRC 在拼接后的流上计算（或在写入发送包缓冲时计算），最后追加 4 字节，可能跨越拍边界。

### 3.4 移位器成本

64 字节旋转器可以按 6 级 2:1 选择器实现（每级 512 bit），共约 3072 个 2:1 选择器；用 LUT6 实现 4:1 选择器时约 3 级 × 512 ≈ 1.5K LUT6，外加 512 bit 暂存寄存器（**粗略估算**）。100G 设计中通常有 3～4 个这样的移位器（RX 负载对齐、TX 拼接、READ 响应拼接等），应在早期资源评估中计入。

## 4. 存储：上下文放在哪里

### 4.1 放置建议

| 数据 | 访问特点 | 规模（见 [RNIC 总体架构](01-rnic-architecture.md) §6） | 建议放置 |
|---|---|---|---|
| QPC 热字段 | 每包读改写，延迟敏感，TX/RX 双方访问 | 64～128 B/QP | 片上（BRAM/URAM）；QP 多时做缓存 |
| QPC 冷字段、头模板、AV | 建链写，每包只读（模板） | 64～128 B/QP | URAM 或片外 + 缓存 |
| 响应资源（READ/ATOMIC） | 仅 READ/ATOMIC/重复请求时访问 | 32 B × 协商深度/QP | URAM 或片外 |
| 定时器表 | 扫描读 + 每个 ACK 写 2 bit | 8 bit/QP | BRAM（宽口，一拍多 QP） |
| MPT | 每次远端访问查一次 | 32～64 B/MR | 缓存 + 片外/主机内存 |
| MTT | 每页一项 | 8 B/页 | 主机内存或片外 + 翻译缓存 |
| 收发包缓冲、读完成重排 | 高带宽流式 | 64～256 KB | BRAM/URAM |

- 器件参考（以器件手册为准）：AMD 器件的 BRAM 为 36 Kb 块、UltraRAM 为 288 Kb 块（72 bit 宽、单时钟，可级联），部分器件/板卡带 HBM；Intel/Altera 器件以 M20K 为主。片外 DDR4/HBM 访问延迟约百 ns 量级，经 PCIe 访问主机内存约 µs 量级。
- 大规模 QP 的经验法则：**片上放"所有 QP 都要的小状态"（定时器表、活跃位、少量热字段），片外/主机放"大而冷"的状态（AV、响应资源、MTT）**。

### 4.2 上下文缓存与软件一致性

- 缓存组织：组相联 + 写回；每个条目带"忙"位，表示有一次读-改-写在流水线中，后续同 QP 访问需等待或走旁路转发；未命中用 MSHR（Miss Status Holding Register）登记，允许其它 QP 命中继续（hit-under-miss）。
- TX 与 RX 都要访问 QPC：用双端口存储或按 QPN 低位分 bank，避免互相阻塞。
- **一致性策略**（**设计建议**）：
  1. 软件不直接写正在被硬件使用的上下文，而是通过命令接口（如 MODIFY_QP）让硬件在缓存中原子更新——硬件是唯一写者；
  2. 若后备存储在主机内存（类似 ICM），这些页在硬件拥有期间软件不得访问；销毁 QP 时先命令硬件写回并失效缓存行，再释放内存；
  3. MR 失效（本地 Invalidate、Send with Invalidate、注销）必须同步失效 MPT/MTT 缓存，并在完成前确认没有在途 DMA 仍使用旧翻译。

### 4.3 复位与异常

- 复位分层：全局复位（上电/FLR）清空全部上下文与在途 DMA；端口级复位（链路断开）只影响该端口的 MAC/PCS——链路断开时一般不直接改变 QP 状态（QP 状态迁移由软件 Modify QP 或传输层检测到的错误触发），在途请求依赖传输定时器超时处理；QP 级"复位"是软件把 QP 改到 RESET，由硬件刷新该 QP 的状态与缓存，而不是硬件复位信号。
- PCIe FLR（Function Level Reset）或热复位时，必须先停止发起新的 DMA、丢弃在途读完成，再清上下文，避免对已被回收的主机内存写入（**设计建议**）。

### 4.4 时序收敛建议

1. **先找环路**：CRC 状态环、QPC 读改写、PSN/ePSN 更新、信用/速率计数器是仅有的几个必须单周期闭合的环，把它们做到 2～3 级 LUT；其余逻辑一律流水。
2. **宽总线扇出**：512 bit 数据通路上的 valid/ready、旁路选择信号扇出巨大，需要寄存器复制；AXI4-Stream 的 ready 反压链用寄存器切片（skid buffer）切断。
3. **多 SLR 器件**：PCIe 硬核与以太网硬核往往位于不同 SLR，跨 SLR 的路径要预留流水寄存器并做布局约束（以器件文档为准）。
4. **UltraRAM 级联**：深级联要打开流水寄存器选项，否则读延迟路径成为关键路径。
5. **大型组合比较器**（如逐字节的屏蔽位置比较）改为查表或预计算掩码。
6. **早做 CDC 与时序约束**：异步 FIFO 的格雷码指针加 `ASYNC_REG` 类约束，跨时钟的多位数据只经 FIFO 传递。

## 5. 资源受限器件上的精简实现方案

### 5.1 目标画像

面向中低端器件、10/25G 或单端口 100G、几十到几百个 QP 的点对点加速器互连：硬件实现 RC 的 SEND/WRITE（建议保留 READ）、ICRC、PSN/ACK/NAK、Go-back-N 重传与定时器；QP 建立、地址解析（ARP/ND）、GID 管理、错误恢复交给软核 CPU 或主机软件；上下文全部放片上。

### 5.2 可裁剪项、互通影响与合规风险

| 可裁剪项 | 节省什么 | 对互通的影响 | 触及的条款 |
|---|---|---|---|
| UD/UC 服务（只做 RC） | UD 的 DETH/Q_Key/AV 逐包处理 | 无法在硬件中处理 QP1 的 CM MAD；用 rdma_cm 建链的应用需要软件 QP1 通路，或改用带外交换 QPN/PSN/GID（如 perftest 默认模式） | C17-11（HCA 须支持 UD/RC/UC） |
| IPv6 | 40 B 头、128 bit 地址比较 | 只能用 IPv4 GID 互通 | CA17-1（须同时支持 IPv4/IPv6） |
| READ | 响应方 READ 引擎、响应资源 | 依赖 READ 的应用（如存储协议）无法工作 | C17-12、C10-83 |
| ATOMIC | 原子执行单元与结果缓存 | 应查询到"不支持原子"并据此运行 | 可选（o10-40 等），无违规 |
| SRQ、XRC、MW、APM、多播、RoCE v1 互通 | 对应上下文与逻辑 | 依赖这些特性的应用不可用 | 均为可选 |
| 端到端信用（一律回"无效"11111） | 信用计算与 LSN 逻辑 | 对端可以"随意发送"（C9-153），一般无碍 | C17-15.2.1、C9-150.2.1（非 SRQ 的 HCA 接收队列须产生信用） |
| RNR NAK | 小 | 不建议裁剪：接收 WQE 不足时只能报错或丢弃，导致连接出错或反复超时 | Figure 90 中 RC 的 RNR NAK 为可选手段 |
| 静态速率控制 | 每 QP 限速器 | 快端口压慢端口时只能依赖 PFC | C17-4 |
| 拥塞控制 RCM | NP/RP 逻辑 | 不支持时发出报文 ECN 必须为 00（CA17-5/14）；建议至少实现 NP（收到 CE 回 CNP），让对端降速 | 可选；但未实现时须满足 CA17-5/14 |
| 内部环回 | TX→RX 旁路 | 本机 QP 间通信失败 | C17-18 |
| 多 SGE | SGE 遍历与 PSN↔偏移重定位 | 只支持单 SGE 的应用（perftest 默认单 SGE） | 无直接条款，影响 Verbs 能力上报 |
| MTT（只支持物理连续缓冲） | 翻译缓存与页表 | 用户态普通 malloc 缓冲无法注册；需大页/预留连续内存 | 内存保护仍须满足 C10-56/57 |
| 多 PMTU | 分段器配置 | 至少支持一种 C17-14 规定的 MTU 集合，并与对端协商一致 | C17-14 |

> [!WARNING]
> 精简方案只能称为"与 RoCEv2 线上格式兼容的 RC 子集"，**不能宣称符合 Annex A17**（CA17-25 把第 9 章与第 17 章的大量条款纳入了合规范围）。与商用 RNIC 或 Linux 互通前，应在建链参数（PMTU、未完成 READ 数、重试次数、超时）上主动与对端对齐，并在验证计划中覆盖被裁剪特性的"优雅拒绝"路径（见 [验证与互通测试](06-verification-and-interop.md)）。

### 5.3 软件辅助的分工

| 功能 | 放在软核/主机 | 硬件提供的接口 |
|---|---|---|
| QP1 / CM、rdma_cm | 软件实现 MAD 解析与状态机 | DestQP=1 报文上送队列；软件构造报文的注入队列 |
| ARP/ND、GID 表 | 软件维护 | GID 表、DMAC 写入接口 |
| QP 状态迁移 | 软件命令 | 命令邮箱，硬件原子更新 QPC |
| 错误恢复与刷新 | 软件可逐个生成 Flushed CQE | QP 错误事件、SQ/RQ 指针读取 |
| 统计与诊断 | 软件读取 | 计数器寄存器 |

## 6. 公开参考实现对比

| 实现 | 性质 | 开源/许可 | 公开资料中的要点 | 本文未能核实的部分 |
|---|---|---|---|---|
| AMD ERNIC（PG332） | 商用 RoCEv2 软 IP | 否，需许可证（RecoNIC 说明可通过 AMD 大学计划申请） | 按 RoCEv2 标准设计；原定位为嵌入式 RDMA 软 IP，控制面由 Arm 或 MicroBlaze 处理；2 个 AXI4-Stream 收发接口 + 5 个 AXI4-MM 接口（取 WQE、负载、写 CQ）；在 RecoNIC 中支持 READ/WRITE/SEND/带立即数的 WRITE 与 SEND/SEND with Invalidate〔外部: RecoNIC 论文 arXiv:2312.06207〕 | QP 数上限、IPv6、Atomic、UD 等以 PG332 为准（未核实） |
| AMD RecoNIC | 开源 SmartNIC 平台 | 是（MIT），RDMA 引擎为 ERNIC | 基于 OpenNIC shell、QDMA、100G MAC，Alveo U250；QP 与缓冲可放主机内存或板载内存；提供 Lookaside/Streaming 计算块〔外部: github.com/Xilinx/RecoNIC〕 | — |
| ETH Zürich fpga-network-stack | 开源网络栈（HLS + RTL） | 是（BSD-3-Clause） | TCP/IP、RoCEv2、UDP，10–100G；RoCE 数据通路为 512 bit AXI4-Stream，ICRC 为 HDL 实现；README 称可与商用 NIC 互通、可达 100 Gb/s；最大 QP 数为编译参数 `FNS_ROCE_STACK_MAX_QPS`（README 中给出的值为 500）；QP 连接信息需由主机带外交换后写入〔外部: github.com/fpgasystems/fpga-network-stack〕 | 原子操作、UD 支持情况未核实 |
| ETH Zürich Coyote（网络栈 BALBOA） | 开源 FPGA shell | 是（MIT，驱动 GPL v2） | README 列出"100G RoCE-v2 compatible RDMA, TCP/IP and collectives"；RecoNIC 论文将其描述为支持 READ/WRITE/SEND 的简化 HLS RoCEv2 实现〔外部: github.com/fpgasystems/Coyote〕 | 完整特性以项目文档与 BALBOA 论文为准 |
| StRoM（EuroSys 2020） | 学术原型 | 网络栈开源于 fpga-network-stack | 在 FPGA RoCEv2 NIC 的数据通路上部署可编程 kernel，就地处理 RDMA 访问的数据〔论文: Sidler et al., StRoM, EuroSys 2020〕 | 具体 kernel 与性能数字以论文为准 |
| Corundum | 开源 FPGA NIC（不含 RDMA） | 是 | 高性能 PCIe DMA 与多队列管理，可借鉴主机接口设计〔外部: Corundum 项目〕 | — |
| Linux rxe（Soft-RoCE） | 内核软件实现 | 是 | 支持 RC/UC/UD 与完整 verbs；逻辑上分为 requester（`rxe_req.c`）、completer（`rxe_comp.c`）、responder（`rxe_resp.c`）三部分；当前主线内核以每 QP 的 `send_task`（`rxe_sender`）与 `recv_task`（`rxe_receiver`）两个任务调度它们，早期版本为三个独立任务〔业界实践: Linux `drivers/infiniband/sw/rxe`，以所用内核源码为准〕 | — |

> [!TIP]
> rxe 的 requester/completer/responder 划分可以直接映射为硬件中的"请求方 TX 引擎 / 请求方 RX 响应处理 / 响应方 RX+TX 引擎"，其源码是理解规范语义细节（如 PSN 比较、重复请求、ICRC 屏蔽）的最佳可执行参考；但它按 skb 逐包处理、使用每 QP 内核定时器，不代表硬件的流水与定时器结构。

## 7. 要点回顾

- 100G 用 512 bit @ 322 MHz 可覆盖最坏拍对齐；400G 必须用每拍多包的分段接口；25G 的 390 MHz 接口宜尽早加宽降频。
- MAC 选型重点核对 PFC、RS-FEC、FCS 错误标示与发送欠载行为；DMA 选型重点核对 Tag 数、RO 属性控制与非对齐支持。
- 并行 CRC 是线性矩阵运算；512 bit 数据部分扇入最多 271，只有状态部分（扇入 ≤ 21）在反馈环中，应把数据部分流水化。
- "64 bit 全 1"前缀可由"寄存器清零 + IP 头前 4 字节替换为 0xFF、更前字节替换为 0x00"实现，无需对齐移位；屏蔽字段只出现在前两拍。
- 尾拍用"补零 + 回卷"处理，6 级常量小矩阵即可；接收侧可用余式（寄存器 0xDEBB20E3，取反后 0x2144DF1C）判定。
- 头部总长恒 ≡ 2 (mod 4)，但负载对齐任意，仍需字节旋转器；每个约 1.5K LUT6 量级。
- 片上放小而热的状态，片外/主机放大而冷的状态；软件经命令接口修改上下文，硬件是唯一写者。
- 精简方案可以互通但不能宣称合规；被裁剪特性要能被对端/应用正确识别为"不支持"。

## 自测

<details><summary>Q1：为什么 512 bit 并行 CRC 不必在一个周期内完成全部 XOR，而状态部分必须？</summary>

答：s_next = A·s ⊕ B·d 中，B·d 只依赖当拍数据，与历史无关，可以提前若干拍流水计算；A·s 依赖上一拍的结果，处在反馈环中，必须一拍闭合。幸运的是 A·s 每位扇入只有约 21，2 级 LUT6 就能完成。
</details>

<details><summary>Q2：IPv4 带 1 层 VLAN 的帧，送入 CRC 前第 0～17 字节应如何替换？</summary>

答：IP 头起点为 18。第 0～13 字节替换为 0x00，第 14～17 字节替换为 0xFF，寄存器初值为 0；这与"初值 0xFFFFFFFF + 先输入 8 个 0xFF 再从 IP 头开始"等价。此外还要替换 TOS（19）、TTL（26）、校验和（28–29）、UDP 校验和（44–45）与 BTH 第 4 字节（50）。
</details>

<details><summary>Q3：最后一拍只有 5 个有效字节时，"补零回卷"需要回卷多少字节？用二进制分解需经过哪几级？</summary>

答：W−k = 64−5 = 59 字节；59 = 32 + 16 + 8 + 2 + 1，经过回卷 32、16、8、2、1 字节这 5 级（4 字节一级旁路）。
</details>

<details><summary>Q4：精简实现只支持 IPv4、只做 RC，能否与 Linux rxe 跑通 ib_write_bw？能否宣称符合 Annex A17？</summary>

答：只要双方使用 IPv4 GID、带外交换 QP 信息（perftest 默认模式不依赖 CM），通常可以跑通；若使用 rdma_cm（-R）则需要 QP1/CM 的软件通路。不能宣称符合 Annex A17：缺 IPv6 违反 CA17-1，缺 UD/UC 违反 C17-11。
</details>

<details><summary>Q5：软件想修改一个活跃 QP 的超时参数，为什么不应直接改写主机内存中的上下文后备存储？</summary>

答：硬件可能在片上缓存中持有该 QPC 的较新副本并会写回，直接改写后备存储会被覆盖或与缓存不一致。应通过命令接口让硬件在缓存中原子修改（硬件是唯一写者），或先让硬件写回并失效该缓存行。
</details>

## 参考

- 规范：Vol1 §7.8.1、§7.8.4（p.218–231，Table 28/33）；§9.7.6.1.3（p.362–365）；§17.2（p.1188–1203）；§A17.3（p.1931–1936）；§A17.7（p.1944–1945）；§A17.9.1（p.1946）；§10.6（p.501–536）。
- 外部：〔外部: AMD PG203（UltraScale+ 100G Ethernet Subsystem）及 BRAM/UltraRAM 器件手册，以文档为准〕；〔外部: IEEE 802.3 Clause 91 RS-FEC、IEEE 802.1Qbb PFC〕；〔外部: PCI Express Base Specification〕；〔外部: RecoNIC，github.com/Xilinx/RecoNIC 与 arXiv:2312.06207〕；〔外部: fpga-network-stack，github.com/fpgasystems/fpga-network-stack〕；〔外部: Coyote，github.com/fpgasystems/Coyote〕；〔论文: D. Sidler et al., "StRoM: Smart Remote Memory", EuroSys 2020〕；〔外部: Corundum 开源 NIC 项目〕；〔业界实践: Linux rxe `rxe_icrc.c`、`rxe_qp.c`、`rxe_loc.h`〕；〔外部: Scapy `scapy.contrib.roce`（用于交叉验证 ICRC）〕。
- 本库相关：[发送数据通路](02-tx-datapath.md)、[接收数据通路](03-rx-datapath.md)、[可靠性引擎](04-reliability-engine.md)、[验证与互通测试](06-verification-and-interop.md)、[ICRC 计算](../01-packet-format/06-icrc.md)、[无损以太网与 PFC](../05-congestion-control-and-qos/01-lossless-ethernet-pfc.md)。
