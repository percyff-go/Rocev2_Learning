# ICRC 计算（Invariant CRC）

> **摘要**：不变量 CRC（Invariant CRC, ICRC）是 RoCEv2 报文中覆盖"端到端不变字段"的 32 bit CRC，它弥补了以太网 FCS 逐跳重算、无法发现设备内部损坏的缺陷。本文给出 IB / RoCE v1 / RoCEv2 三种 ICRC 规则的对比、完整的屏蔽字段清单、CRC 参数与字节/位序，提供一个用规范 §7.8.4 两个样例验证通过的 Python 参考实现，并讨论并行 CRC、屏蔽、对齐、尾拍、残差校验等硬件实现要点和常见错误。
>
> **规范出处**：Vol1 §7.8.1（p.218–219）；§7.8.4（p.221–231）；§5.2.14（p.179）；§9.2 Figure 67（p.256–257）；§A16.3.2（p.1913）；§A17.3.3–§A17.3.4（p.1935–1936）
>
> **前置阅读**：[01-frame-layout.md](01-frame-layout.md)、[02-ethernet-ip-udp.md](02-ethernet-ip-udp.md)、[03-bth.md](03-bth.md)
>
> **关键词**：ICRC, FCS, CRC-32, 0x04C11DB7, 屏蔽字段, 64 bit 全 1 前缀, 反射 CRC, zlib, 并行 CRC, 残差, 0xDEBB20E3

## 1. ICRC 是什么，为什么需要它

- ICRC 覆盖报文中从源到目的**不会被改变**的所有字段；所有 IBA 传输包都必须携带并按 §7.8.1 计算（C7-47、C7-48）〔Vol1 §7.8.1, p.218〕。
- RoCEv2 中 ICRC 提供"超出以太网 FCS 覆盖范围"的保护，因为 FCS 通常在网络中逐跳更新〔Vol1 §A17.3.3, p.1935〕。
- 不符合 ICRC 规则的报文**静默丢弃**（CA17-24）〔Vol1 §A17.3.4, p.1936〕。

| 对比项 | ICRC | 以太网 FCS |
|---|---|---|
| 覆盖范围 | 64 bit 全 1 + IP 头 … 负载/pad（可变字段置 1） | DMAC … ICRC（含以太网填充） |
| 生成者 | 源端 RNIC，一次 | 每个发出该帧的设备（逐跳） |
| 校验者 | 目的端 RNIC | 每个接收该帧的设备（逐跳） |
| 中途路由器改 TTL、MAC、VLAN 后 | 不变 | 重新计算 |
| 能发现设备内部（缓存/总线）损坏 | 能 | 不能 —— 路由器会为损坏的数据算出新的正确 FCS |
| 多项式与位序 | CRC-32，0x04C11DB7 | 相同 |

## 2. 三种封装的 ICRC 规则对比

| 项目 | IB 本地包（无 GRH） | IB 全局包（有 GRH） | RoCE v1（Annex A16） | RoCEv2（Annex A17） |
|---|---|---|---|---|
| 计算起点 | LRH 第 0 字节 | LRH 第 0 字节 | 64 bit 全 1（代替 LRH） | **64 bit 全 1** |
| 链路层头 | LRH 参与，**VL 置 1** | LRH **整体置 1** | MAC 头不参与 | MAC 头（含 VLAN）不参与 |
| 网络层头 | — | GRH 参与，**TClass、FlowLabel、HopLmt 置 1** | 同左（GRH） | IPv4：**TOS、TTL、Header Checksum 置 1**；IPv6：**Traffic Class、Flow Label、Hop Limit 置 1** |
| UDP 头 | — | — | — | 参与，**Checksum 置 1** |
| BTH | **Resv8a 置 1** | 同左 | 同左 | 同左 |
| 扩展头、负载、pad | 参与 | 参与 | 参与 | 参与 |
| 依据 | §7.8.1, p.218 | §7.8.1, p.218–219 | §A16.3.2, p.1913 | CA17-22, p.1935–1936 |

两个推论：
1. **RoCE v1 的 ICRC 与 IB 全局包完全相同**：IB 全局包中 LRH 本来就整体置 1，而 RoCE v1 规定"虽然没有 LRH，仍先计入 64 bit 全 1，MAC 头不参与"〔Vol1 §A16.3.2, p.1913〕。
2. **RoCEv2 的 64 bit 全 1 前缀**正是为了与 RoCE v1 中"假 LRH"等价（CA17-22 脚注 61）；之后用 IP/UDP 头代替 GRH 进入 CRC，并把 UDP Checksum 也视为可变字段。

> [!NOTE]
> §7.8.1 还写道 "The CRC always starts with LRH:LVer bit 0"〔Vol1 p.219〕。这与"按字节、每字节 LSB 先入"一致：LRH 第 0 字节的高 4 位是 VL（被置 1），低 4 位是 LVer，LSB 先入时第一个进入 CRC 的正是 LVer 的 bit 0。

## 3. RoCEv2 屏蔽字段清单

"屏蔽"指**计算时把该字段替换为全 1**，而不是跳过；报文中发送的仍是原值。

| 字段 | IP 版本 | 相对 IP 头的偏移（字节） | 帧偏移（无 VLAN / 单 VLAN） | 宽度 |
|---|---|---|---|---|
| TOS（DSCP + ECN） | IPv4 | 1 | 15 / 19 | 8 bit |
| TTL | IPv4 | 8 | 22 / 26 | 8 bit |
| Header Checksum | IPv4 | 10–11 | 24–25 / 28–29 | 16 bit |
| UDP Checksum | IPv4 | 26–27 | 40–41 / 44–45 | 16 bit |
| BTH Resv8a（F、B、Resv6） | IPv4 | 32 | 46 / 50 | 8 bit |
| Traffic Class（DSCP + ECN） | IPv6 | 0 的低 4 位 + 1 的高 4 位 | 14–15 / 18–19 | 8 bit |
| Flow Label | IPv6 | 1 的低 4 位 + 2–3 | 15–17 / 19–21 | 20 bit |
| Hop Limit | IPv6 | 7 | 21 / 25 | 8 bit |
| UDP Checksum | IPv6 | 46–47 | 60–61 / 64–65 | 16 bit |
| BTH Resv8a | IPv6 | 52 | 66 / 70 | 8 bit |

IPv6 的 Traffic Class 与 Flow Label 合起来正好是第 0 字节低 4 位加第 1–3 字节，所以屏蔽后 IPv6 头前 4 字节恒为 `6F FF FF FF`。

**不屏蔽、按原值参与 ICRC 的字段**：IP Version/IHL、Total Length / Payload Length、Identification、Flags/Fragment Offset、Protocol / Next Header、源/目的 IP 地址；UDP 源端口、目的端口、Length；BTH 除第 4 字节以外的全部字段（**包括 Resv7**）；所有扩展头、负载和 PadCnt 指示的 pad 字节。

**完全不进入 ICRC 的部分**：以太网头（含 VLAN 标签）、以太网填充、FCS，以及 ICRC 字段本身。

## 4. CRC 参数、字节序与位序

### 4.1 参数

| 参数 | 取值 | 依据 |
|---|---|---|
| 宽度 | 32 bit | §7.8.1 |
| 生成多项式 | 0x04C11DB7（与以太网相同）；反射形式 0xEDB88320 | §7.8.1, p.219 |
| 初值 | 0xFFFFFFFF | §7.8.1 步骤 1 |
| 输入位序 | 按字节顺序处理，**每字节最低位先进入 CRC**（即"反射输入"） | §7.8.1 步骤 2 |
| 输出 | 结果按位取反 | §7.8.1 步骤 3 |
| 发送顺序 | 从余式最高次项系数开始发送，与以太网 CRC 一致 | §7.8.1 步骤 4 |
| 等价算法 | 与以太网 FCS、zlib `crc32()`、`binascii.crc32()` 完全相同（"CRC-32/ISO-HDLC"，校验值 `crc32("123456789") = 0xCBF43926`） | 〔外部: ISO/IEC 3309、zlib〕 |

### 4.2 三种记法

同一个 ICRC 在不同场合有三种写法，混淆它们是互通失败最常见的原因之一。以规范 Table 28 的样例为例：

| 记法 | 值 | 说明 |
|---|---|---|
| zlib / binascii 返回值 | `0x5AB72596` | `zlib.crc32(屏蔽后的字节流)` |
| **线上字节**（ICRC 字段的 4 个字节，按发送顺序） | `96 25 B7 5A` | = `struct.pack('<I', zlib 值)`，即 zlib 值的**小端**存放 |
| 规范记法（"Generated ICRC is 0x9625B75A"） | `0x9625B75A` | = 线上 4 字节按**大端**读出 |

规范 Figure 57 用串行参考电路说明了这一关系：余式（Remainder）取反后，**每个字节内部做位反转**映射到 ICRC 字段（余式 bit 31…24 → ICRC bit 24…31，依此类推），ICRC 字段再按大端发送〔Vol1 §7.8.4.1, p.221–222〕。字节内位反转与"字节内 LSB 先入"相互抵消后，最终效果就是"zlib 值小端存放"。这与以太网 FCS 的存放方式完全一样。

〔业界实践: Linux rxe `rxe_icrc.c`〕 软件实现 rxe 用 `crc32_le()`（反射 CRC，不做首尾取反）以 `0xdebb20e3` 为初值依次处理屏蔽后的头部和负载，最后 `*icrcp = ~crc` 以主机字节序写入 ICRC 位置 —— 在小端主机上正是"zlib 值小端存放"。`0xdebb20e3` 即"初值 0xFFFFFFFF 处理 8 个 0xFF 字节后的 CRC 状态"（§6.3 已验证）。

## 5. Python 参考实现

### 5.1 代码

下面的实现同时给出逐位的反射算法、按 Figure 57 逐位实现的"规范直译"版本、三种封装的屏蔽函数，并以 §7.8.4 的两个样例做自检。样例字节流取自 Table 27（本地包）和 Table 32（全局包）〔Vol1 p.225、p.228–229〕；屏蔽函数的输出已在验证脚本中与 Table 28、Table 33 逐字节比对一致。

```python
#!/usr/bin/env python3
"""InfiniBand / RoCEv2 ICRC 参考实现（教学用）。

依据：IBTA Vol1 §7.8.1（ICRC 定义）、§A16.3.2（RoCE）、CA17-22（RoCEv2）。
- 多项式 0x04C11DB7，初值 0xFFFFFFFF，每字节 LSB 先入（= 反射算法），结果取反；
- 返回值一律是"线上字节顺序"的 4 个字节（紧跟在负载/pad 之后发送）。
"""
import struct
import zlib

POLY_REFLECTED = 0xEDB88320  # 0x04C11DB7 按位反转


def crc32_update(state: int, data: bytes) -> int:
    """逐位实现的反射 CRC-32 状态更新（不含初值装载与最终取反）。"""
    for byte in data:
        state ^= byte
        for _ in range(8):
            state = (state >> 1) ^ (POLY_REFLECTED if state & 1 else 0)
    return state


def icrc_bytes(masked: bytes) -> bytes:
    """对"已屏蔽"的字节流计算 ICRC，返回线上字节顺序。"""
    state = crc32_update(0xFFFFFFFF, masked)
    return struct.pack("<I", state ^ 0xFFFFFFFF)


def icrc_field_literal(masked: bytes) -> int:
    """按 §7.8.4 Figure 57 串行参考电路逐位实现，返回规范记法的 ICRC 字段值。

    非反射（MSB 先移位）LFSR，但每个字节仍从 LSB 开始送入；
    结果取反后，按 Figure 57 在每个字节内做位反转映射到 ICRC 字段，再按大端发送。
    """
    s = 0xFFFFFFFF
    for byte in masked:
        for i in range(8):                      # 字节内 LSB 先入
            fb = ((s >> 31) ^ (byte >> i)) & 1
            s = (s << 1) & 0xFFFFFFFF
            if fb:
                s ^= 0x04C11DB7
    rem = s ^ 0xFFFFFFFF                        # Remainder: bit31 = x^31 系数
    field = 0
    for k in range(4):                          # 字节内位反转（31..24 -> 24..31 ...）
        b = (rem >> (8 * k)) & 0xFF
        field |= int(f"{b:08b}"[::-1], 2) << (8 * k)
    return field


# ---------- 三种封装的屏蔽规则 ----------
def mask_ib_local(pkt: bytes) -> bytes:
    """IB 本地包：LRH(8B) + BTH + ...（不含 ICRC/VCRC）。"""
    b = bytearray(pkt)
    b[0] |= 0xF0                  # LRH:VL（第 0 字节高 4 位）
    b[8 + 4] = 0xFF               # BTH:Resv8a（F/B/Resv6 所在字节）
    return bytes(b)


def mask_ib_global(pkt: bytes) -> bytes:
    """IB 全局包 / RoCEv1：LRH(8B，RoCEv1 中不存在，按全 1 处理) + GRH(40B) + BTH + ..."""
    b = bytearray(pkt)
    b[0:8] = b"\xff" * 8          # 整个 LRH
    b[8] |= 0x0F                  # GRH:TClass 高 4 位
    b[9:12] = b"\xff" * 3         # GRH:TClass 低 4 位 + FlowLabel
    b[15] = 0xFF                  # GRH:HopLmt
    b[48 + 4] = 0xFF              # BTH:Resv8a
    return bytes(b)


def mask_rocev2(ip_datagram: bytes) -> bytes:
    """RoCEv2：输入为 IP 头首字节 ~ ICRC 前最后一个字节（不含以太网头、ICRC、FCS）。"""
    b = bytearray(ip_datagram)
    version = b[0] >> 4
    if version == 4:
        udp = (b[0] & 0x0F) * 4   # RoCEv2 要求 IHL=5（CA17-3），即 20
        b[1] = 0xFF               # TOS = DSCP + ECN
        b[8] = 0xFF               # TTL
        b[10:12] = b"\xff\xff"    # Header Checksum
    elif version == 6:
        udp = 40
        b[0] |= 0x0F              # Traffic Class 高 4 位
        b[1:4] = b"\xff" * 3      # Traffic Class 低 4 位 + Flow Label
        b[7] = 0xFF               # Hop Limit
    else:
        raise ValueError("not an IPv4/IPv6 datagram")
    b[udp + 6:udp + 8] = b"\xff\xff"   # UDP Checksum
    b[udp + 8 + 4] = 0xFF               # BTH:Resv8a
    return b"\xff" * 8 + bytes(b)       # 先送入 64 bit 全 1


def rocev2_icrc(ip_datagram: bytes) -> bytes:
    return icrc_bytes(mask_rocev2(ip_datagram))


# ---------- 自检：§7.8.4 的两个样例（Table 27 / Table 32 的未屏蔽字节流） ----------
TABLE27 = bytes.fromhex(
    "7012375c000e17d2"                          # LRH
    "0a202487 0087b1b3 000dec2a"                # BTH
    "01710a1c015d4002 38f27a05 0000000e"        # RETH
    "bb884d85fd5cfba4728bc0690ed4 0000")        # 14B 负载 + 2B pad
TABLE32 = bytes.fromhex(
    "7013375c001817d2"                          # LRH
    "60000000 00300010"                         # GRH: IPVer/TClass/FlowLabel, PayLen/NxtHdr/HopLmt
    "00000000000001250000000000000026"          # SGID
    "00000000000001170000000000000096") + TABLE27[8:]  # DGID + 与本地样例相同的 BTH/RETH/负载

if __name__ == "__main__":
    for name, masked, expect in (("Table 27/28", mask_ib_local(TABLE27), 0x9625B75A),
                                 ("Table 32/33", mask_ib_global(TABLE32), 0xB67D1ED1)):
        wire = icrc_bytes(masked)
        assert wire == struct.pack(">I", expect)                  # 线上字节 = 规范值按大端
        assert wire == struct.pack("<I", zlib.crc32(masked))      # 与 zlib/binascii 一致
        assert icrc_field_literal(masked) == expect               # 与 Figure 57 逐位电路一致
        print(f"{name}: ICRC wire bytes = {wire.hex(' ').upper()}  (spec 0x{expect:08X})  OK")
```

运行结果（Python 3.12）：

```text
Table 27/28: ICRC wire bytes = 96 25 B7 5A  (spec 0x9625B75A)  OK
Table 32/33: ICRC wire bytes = B6 7D 1E D1  (spec 0xB67D1ED1)  OK
```

两个样例都与规范给出的结果一致，并且逐位反射算法、Figure 57 直译算法、`zlib.crc32` 三者相互一致。

> [!NOTE]
> 同样的位序约定也适用于 IB 的 VCRC（CRC-16，多项式 0x100B）：用反射形式 0xD008、初值 0xFFFF、结果取反并小端存放，可以复现 Table 29 的 VCRC `45 FA` 和 Table 34 的 `B1 48`（验证脚本中已核对）。RoCE 不使用 VCRC（CA16-3），这里只作为位序理解的旁证。

### 5.2 RoCEv2/IPv4 示例（与 01-frame-layout.md 的报文一致）

[01-frame-layout.md](01-frame-layout.md) §7 的 RC RDMA WRITE Only 报文，取 IP 头首字节到负载末字节作为输入：

```python
ip_datagram = bytes.fromhex(
    "456a004c2b1a40004011 8bae c0a8010a c0a80114"   # IPv4 头（20 B）
    "c8f112b7 0038 0000"                           # UDP 头（8 B）
    "0a40ffff 000012a7 805a3c01"                   # BTH（12 B）
    "00007f3ac0001000 00021f37 00000010"           # RETH（16 B）
) + b"RoCEv2 ICRC demo"                             # 负载（16 B）
icrc = rocev2_icrc(ip_datagram)
print(icrc.hex(" ").upper())                       # 线上字节
print(f"0x{struct.unpack('>I', icrc)[0]:08X}")     # 规范记法
print(f"0x{zlib.crc32(mask_rocev2(ip_datagram)):08X}")   # zlib 值
```

```text
81 30 D3 F3
0x8130D3F3
0xF3D33081
```

送入 CRC 的屏蔽后字节流（80 字节，方括号标出被替换为全 1 的位置）：

```text
0000  FF FF FF FF FF FF FF FF 45[FF]00 4C 2B 1A 40 00     64 bit 全 1 | Ver/IHL, [TOS], Total Length, ID, Flags
0010 [FF]11[FF FF]C0 A8 01 0A C0 A8 01 14 C8 F1 12 B7     [TTL], Protocol, [Checksum], SIP, DIP | UDP 端口
0020  00 38[FF FF]0A 40 FF FF[FF]00 12 A7 80 5A 3C 01     UDP Length, [UDP Checksum] | BTH（[Resv8a]）
0030  00 00 7F 3A C0 00 10 00 00 02 1F 37 00 00 00 10     RETH
0040  52 6F 43 45 76 32 20 49 43 52 43 20 64 65 6D 6F     负载
```

注意第 0x26–0x27 的 `FF FF` 是 P_Key 本身的值（0xFFFF），不是屏蔽结果。

### 5.3 RoCEv2/IPv6 示例（含 PadCnt）

把 §7.8.4 样例中的 BTH、RETH、14 字节负载和 2 字节 pad 原样放进 RoCEv2/IPv6 封装（Traffic Class 0x6A、Flow Label 0xA1B2C、Hop Limit 64，地址取文档前缀 2001:db8::/32）：

```python
ip6_datagram = bytes.fromhex(
    "66aa1b2c 0038 1140"                                  # Ver 6 | TC 0x6A | FL 0xA1B2C | PayLen 56 | NH 0x11 | HL 64
    "20010db8 00000000 00000000 0000000a"                 # SIP 2001:db8::a
    "20010db8 00000000 00000000 00000014"                 # DIP 2001:db8::14
    "c8f112b7 0038 0000"                                  # UDP
) + TABLE27[8:]                                           # 规范样例的 BTH + RETH + 14 B 负载 + 2 B pad
print(rocev2_icrc(ip6_datagram).hex(" ").upper())         # → ED 82 0C 2C
```

屏蔽后的前 24 字节为 `FF×8 | 6F FF FF FF 00 38 11 FF | 20 01 0D B8 …`：Traffic Class、Flow Label、Hop Limit 都已置 1。把同一个 IP 包放进带 VLAN 标签或不带标签的以太网帧，ICRC 都是 `ED 82 0C 2C`；把 Traffic Class、Flow Label、Hop Limit 改成任意值，ICRC 也不变（脚本已验证）。

### 5.4 不变性实验

对 §5.2 的报文逐个修改字段后重算 ICRC：

```python
def patch(buf, off, val):
    b = bytearray(buf); b[off] = val; return bytes(b)

cases = {"TTL 64→63": (8, 0x3F), "ECN 10→11(CE)": (1, 0x6B), "UDP Checksum": (26, 0x12),
         "BTH FECN": (32, 0x80), "UDP 源端口": (20, 0xC9), "IP Identification": (4, 0x00),
         "BTH Resv7": (36, 0x81), "BTH PSN": (39, 0x02)}
for name, (off, val) in cases.items():
    same = rocev2_icrc(patch(ip_datagram, off, val)) == icrc
    print(f"{name:18s} -> {'不变' if same else '改变'}")
```

| 修改的字段 | ICRC | 原因 |
|---|---|---|
| TTL 64 → 63（路由器每跳递减） | 不变 | 屏蔽 |
| ECN 10 → 11（交换机标记 CE） | 不变 | TOS 整字节屏蔽 |
| UDP Checksum | 不变 | 屏蔽 |
| BTH FECN 位 | 不变 | Resv8a 屏蔽 |
| UDP 源端口 | **改变** | 不屏蔽 —— NAT/PAT 会破坏 RoCEv2 |
| IP Identification | **改变** | 不屏蔽 |
| BTH Resv7 | **改变** | Resv7 参与 ICRC |
| BTH PSN | **改变** | 参与 ICRC |

## 6. 「硬件实现要点」

### 6.1 并行 CRC

CRC 是 GF(2) 上的线性运算：每拍处理 W 位数据时，新状态 `S' = A^W · S ⊕ B_W · D`，其中 `A^W`（32×32）和 `B_W`（32×W）都是常数矩阵，展开后每个输出位就是若干状态位与数据位的异或。

| 数据通路宽度 | 典型用途 | 实现提示 |
|---|---|---|
| 8 B/拍（64 bit） | 10/25G | 单级异或树即可 |
| 16–32 B/拍 | 25G/50G/100G（较高时钟） | 异或树变深，考虑插入一级流水 |
| 64 B/拍（512 bit） | 100G/200G | 每个输出位扇入数百，常拆成"数据贡献 ⊕ 状态贡献"两级流水，或分段计算后用 CRC 合并（combine）公式拼接 |

矩阵可由软件生成：对单位向量逐位运行 §5.1 的 `crc32_update` 即可得到每一列（具体生成方法与资源估算见 [FPGA 实现要点](../06-hardware-implementation/05-fpga-implementation-notes.md)）。

### 6.2 按偏移替换屏蔽字段

解析器给出 `{VLAN 个数, IP 版本}` 后，屏蔽字节在帧中的位置就完全确定（§3 表）。实现上为每拍生成一个"字节屏蔽向量"，CRC 输入 `crc_in[i] = mask[i] ? 8'hFF : data[i]`（IPv6 第一个字节只屏蔽低 4 位）。发送端只修改送入 CRC 引擎的那一份数据，**报文本身保持原值**。

### 6.3 64 bit 全 1 前缀与起点对齐

| 方案 | 做法 | 说明 |
|---|---|---|
| A：预置种子 | 在 IP 头第一个字节处开始计算，初值用"处理完 8 个 0xFF 后的状态"：反射形式 **0xDEBB20E3**，非反射（MSB 先移位）寄存器形式 **0xC704DD7B** | rxe 即用此法；需要处理 IP 头不在拍首的情况（字节使能） |
| B：零状态对齐 | 从帧第 0 字节开始、初值 **0**：IP 头之前的字节全部替换为 0x00，只有紧挨 IP 头的最后 4 个字节替换为 0xFF | 不需要任何移位器，CRC 引擎与帧边界对齐 |

方案 B 成立的原因（推导 + 脚本验证）：零状态下输入零字节，状态保持为 0；从 0 状态输入 4 个 0xFF 后状态为 0xDEBB20E3；而从 0xFFFFFFFF 输入 4 个 0xFF 后状态恰好为 0（初值被抵消），再输入 4 个 0xFF 同样得到 0xDEBB20E3。因此两种方案在 IP 头首字节处的状态完全相同。验证脚本对 L2 头为 14、18、22 字节（无 VLAN、单 VLAN、QinQ）三种情况都得到与参考实现一致的 ICRC。

### 6.4 尾拍与长度

- ICRC 的最后一个输入字节由 **IP 长度**决定（`IP 起点 + IP 包长 − 5`），而不是帧尾；以太网填充必须排除（见 [01-frame-layout.md](01-frame-layout.md) §6）。
- 尾拍通常只有部分字节有效，常见处理方式：
  1. **多套尾拍 CRC**：为每种有效字节数准备一套矩阵并按尾长选择，速度最快但面积大（64 B 通路需要最多 64 套）。
  2. **级联折半**：尾拍按 32/16/8/4/2/1 字节分段，由有效字节数的二进制位决定每段是否参与，只需 log₂W 套矩阵。
  3. **前置零右对齐**：利用"零状态 + 零字节不改变状态"的性质（§6.3），在包头前补 k 个 0x00，使包尾正好落在拍边界上。发送端长度已知；接收端可以从第一拍的 IP 头取得长度，但需要一个起点移位器。脚本已验证补 0–69 个零字节结果不变。

### 6.5 接收端校验：重算比较与残差法

1. **重算比较**：对屏蔽后的字节流计算 ICRC，与报文中 ICRC 位置的 4 个字节比较（需要按 IP 长度定位 ICRC）。
2. **残差法**：把报文中的 ICRC 字节（不屏蔽）接在屏蔽后的字节流后面一起送入同一 CRC 引擎，正确时最终状态（取反之前）恒为常数 **0xDEBB20E3**（非反射寄存器为 0xC704DD7B），即 `zlib.crc32(stream + icrc) == 0x2144DF1C`。残差法不需要单独抽取 ICRC 字段做比较，适合与尾拍处理合并。

```python
assert crc32_update(0xFFFFFFFF, b"\xff" * 8) == 0xDEBB20E3                        # 前缀种子
assert crc32_update(0xFFFFFFFF, mask_rocev2(ip_datagram) + icrc) == 0xDEBB20E3   # 残差
```

> [!NOTE]
> 种子与残差恰好是同一个常数并非巧合：二者都等于"从零状态输入 4 个 0xFF 字节后的状态"。以太网 MAC 用残差法校验 FCS 时使用的也是这个常数。

### 6.6 ICRC 与 FCS 的关系

- 两者由不同的引擎并行计算：FCS 覆盖整帧、不屏蔽任何字节；ICRC 从 IP 头开始并做屏蔽。发送端先算 ICRC 并插入报文，再由 MAC 计算 FCS；接收端 MAC 校验 FCS，RoCE 引擎校验 ICRC。
- **FCS 正确而 ICRC 错误**，说明数据在某台设备内部（缓存、总线、转发芯片）被破坏，或者两端 ICRC 实现不一致 —— 因为路由器改写 TTL/MAC 后会为（可能已损坏的）数据重新生成正确的 FCS。排障时这两个计数器要分开看，见 [排障指南](../07-engineering-practice/05-troubleshooting.md)。
- 一个有趣的代数性质（推导 + 脚本验证）：两者使用同一多项式和位序，且 ICRC 紧挨在 FCS 覆盖区的末尾（其后只可能有全零的以太网填充），所以当 ICRC 正确时，**修改任何被 ICRC 覆盖且未屏蔽的字节并重算 ICRC，FCS 都不会改变**（对 296 个随机报文的测试无一例外）。也就是说，此时 FCS 只取决于以太网头和那些被屏蔽字段的实际取值。调试构造报文时看到"改了负载、FCS 却没变"不必惊讶。

### 6.7 提交时机

ICRC 结果要到包尾才能得到，而解析、查表等工作在包头就已开始。所有**不可撤销的动作**——推进期望 PSN、调度 ACK/NAK、产生 CQE、执行原子操作、生成 READ 响应、执行远程失效——都必须等 ICRC 通过后才提交。对 RDMA WRITE / SEND 的负载，直通（cut-through）设计可能在 ICRC 结果出来之前就把数据写入了目标内存：由于 ICRC 失败的包被静默丢弃、PSN 不推进，请求方会重传覆盖同一位置，而应用在完成之前本就不能依赖缓冲区内容（o9-20、C9-33）〔Vol1 §9.5, p.292〕，所以这在协议语义上是可以接受的（工程推导）；原子操作和 READ 请求则必须等待校验结果。详见 [接收通路](../06-hardware-implementation/03-rx-datapath.md)。

## 7. 常见错误清单

| # | 错误 | 典型现象 | 正确做法 |
|---|---|---|---|
| 1 | 字节内按 MSB 先送入 CRC | 与任何对端都不匹配 | 反射算法；或非反射 LFSR 但每字节 LSB 先入（§5.1 的 `icrc_field_literal`） |
| 2 | ICRC 4 字节顺序写反 | 数值"看起来对"，线上字节颠倒 | 线上字节 = `pack('<I', zlib 值)` = `pack('>I', 规范记法值)` |
| 3 | 漏掉 64 bit 全 1 前缀，或把前缀当作"初值改为全 1" | 全部失败 | 先输入 8 个 0xFF，或直接用种子 0xDEBB20E3 |
| 4 | 忘记屏蔽 UDP Checksum | 自发自收测试正常（自己总发 0），与正确实现互通时全部失败 | 自测通过不代表正确，必须用规范样例和第三方实现交叉验证 |
| 5 | IPv4、IPv6 屏蔽字段混用 | 某一种 IP 版本互通失败 | IPv4：TOS、TTL、Header Checksum；IPv6：Traffic Class、Flow Label、Hop Limit |
| 6 | 只屏蔽 DSCP 不屏蔽 ECN | 平时正常，网络一拥塞（ECN 被标记为 CE）就丢包，表现为拥塞时吞吐骤降 | 屏蔽整个 TOS / Traffic Class 字节 |
| 7 | IPv6 Traffic Class 跨字节处理错误 | IPv6 失败 | 第 0 字节低 4 位 + 第 1 字节高 4 位 |
| 8 | 漏屏蔽 BTH Resv8a，或误屏蔽了 Resv7 | 与正确实现互通失败 | 只屏蔽 BTH 第 4 字节 |
| 9 | 把屏蔽后的 0xFF 写回了发送报文 | 对端看到 TOS=0xFF、F/B=1 等异常 | 只修改送入 CRC 引擎的副本 |
| 10 | PadCnt 的 pad 字节未计入 ICRC | 负载长度非 4 的倍数时失败 | pad 参与 ICRC；pad 内容规范未规定，发送端应填 0 |
| 11 | 用帧尾定位 ICRC，或把以太网填充算进 ICRC | 只有短包（零长度 SEND、剥离 VLAN 后的小包）失败 | 按 IP 长度定位 ICRC |
| 12 | VLAN 存在时屏蔽偏移没有更新 | 带 VLAN 时失败、不带时正常 | 屏蔽位置相对 IP 头计算 |
| 13 | 误把 UDP 源端口或 Length 当作可变字段屏蔽 | 与正确实现互通失败 | 只有 UDP Checksum 被屏蔽 |
| 14 | 把 ICRC 当作 IP/UDP 之外的"尾部"而没有计入 IP Total Length / UDP Length | 对端长度校验失败或 ICRC 定位错误 | 长度字段必须包含 4 字节 ICRC（CA17-6、CA17-15、CA17-21） |

## 8. 要点回顾

- ICRC 是端到端 CRC-32（0x04C11DB7，初值全 1，字节内 LSB 先入，结果取反），与以太网 FCS / zlib `crc32` 算法完全相同。
- RoCEv2 规则（CA17-22）：先输入 64 bit 全 1，再从 IP 头首字节输入到 ICRC 前最后一个字节；IPv4 屏蔽 TOS/TTL/Header Checksum，IPv6 屏蔽 Traffic Class/Flow Label/Hop Limit，另外屏蔽 UDP Checksum 和 BTH Resv8a。
- UDP 源端口、IP 地址、Identification、BTH Resv7、pad 都参与 ICRC；以太网头、以太网填充、FCS 不参与。
- 线上 ICRC 字节 = zlib 值小端存放 = 规范记法值大端存放；规范样例 0x9625B75A、0xB67D1ED1 已用参考实现复现。
- 硬件：并行矩阵 CRC；按偏移把屏蔽字节替换为 0xFF；前缀可用种子 0xDEBB20E3 或"零状态 + 4 个 0xFF"对齐技巧；ICRC 位置按 IP 长度确定；接收端可用残差 0xDEBB20E3 判定。
- FCS 正确而 ICRC 错误意味着设备内部损坏或实现不一致；所有不可撤销动作都要等 ICRC 通过后提交。

## 自测

<details><summary>Q1：某实现计算出 zlib.crc32(屏蔽字节流) = 0x12345678，ICRC 字段应按什么顺序发送？规范记法是多少？</summary>

答：按小端发送：78 56 34 12。规范记法（4 字节按大端读出）是 0x78563412。
</details>

<details><summary>Q2：为什么 RoCEv2 要先输入 64 bit 全 1，而不是直接从 IP 头开始？</summary>

答：为了与 RoCE v1 和 IB 全局包保持同一计算框架：IB 全局包中整个 LRH（8 字节）被替换为全 1 参与计算，RoCE v1 没有 LRH 但仍计入 64 bit 全 1（§A16.3.2），RoCEv2 沿用这一"假 LRH"使计算方式与 RoCE v1 等价（CA17-22 脚注 61）。数学上它等价于把初值从 0xFFFFFFFF 换成 0xDEBB20E3。
</details>

<details><summary>Q3：交换机把一个 RoCEv2/IPv6 报文的 ECN 从 10 改成 11、Hop Limit 减 1、同时剥掉了 VLAN 标签，接收端 ICRC 是否仍然正确？FCS 呢？</summary>

答：ICRC 仍然正确：Traffic Class（含 ECN）和 Hop Limit 被屏蔽，VLAN 标签根本不在 ICRC 范围内。FCS 由该交换机按修改后的帧重新生成，所以也是正确的。
</details>

<details><summary>Q4：一个自研 RNIC 与自己对测完全正常，与商用网卡互通时所有报文 ICRC 都失败，但 UDP Checksum 一直是 0。最可能的原因是什么？</summary>

答：很可能忘记屏蔽 UDP Checksum（或屏蔽了其它不该屏蔽的字段、字节序写反）。双方都用同一错误实现时结果自洽，自测发现不了；应先用本文参考实现与规范样例核对，再用抓包对比逐字节检查屏蔽后的字节流。
</details>

<details><summary>Q5：接收端用残差法校验时，应把什么送入 CRC 引擎？正确时最终状态是多少？</summary>

答：送入"64 bit 全 1 + 屏蔽后的 IP 头 … pad + 报文中原样的 4 字节 ICRC"，以 0xFFFFFFFF 为初值（或从 IP 头开始、以 0xDEBB20E3 为初值）；正确时取反之前的状态恒为 0xDEBB20E3（反射形式），等价于 `zlib.crc32(...) == 0x2144DF1C`。
</details>

## 参考

- 规范：Vol1 §5.2.14（p.179）；§7.8.1（p.218–219）、Figure 56（p.219）；§7.8.4（p.221–231）：Figure 57（p.221）、Table 23–29（p.223–226）、Table 30–34（p.227–231）；§9.2 Figure 67（p.256–257）；§9.5（p.291–292）；§A16.3.2（p.1913）；§A17.3.3–§A17.3.4（p.1935–1936）。
- 外部：IEEE 802.3（以太网 FCS）；ISO/IEC 3309（HDLC CRC-32）；zlib / Python `binascii.crc32`。
- 业界实践：Linux 内核 `drivers/infiniband/sw/rxe/rxe_icrc.c`（种子 0xdebb20e3、屏蔽方式与写入方式）。
- 相关文件：[01-frame-layout.md](01-frame-layout.md)、[02-ethernet-ip-udp.md](02-ethernet-ip-udp.md)、[03-bth.md](03-bth.md)、[发送数据通路](../06-hardware-implementation/02-tx-datapath.md)、[接收通路](../06-hardware-implementation/03-rx-datapath.md)、[验证与互通](../06-hardware-implementation/06-verification-and-interop.md)。
