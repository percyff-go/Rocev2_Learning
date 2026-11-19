# RoCEv2 对 Verbs 的修改与编程实践（RoCEv2 Verbs Modifications and Programming Practice）

> **摘要**：Annex A17 要求 RoCEv2 HCA 采纳 §A17.5 对 Verbs 的全部修改（CA17-30），核心是“以 GID 取代 LID 的地址向量”和“去掉只属于 IB 链路层与子网管理的能力”。本文先逐条整理 §A17.5（及其沿用的 §A16.9）对各 Verb 的修改，再给出一个结构完整、可编译的 libibverbs RC 示例（含 RoCEv2 GID 选择与 TCP 带外交换）、UD 片段与 librdmacm 简化流程，最后汇总常见编程陷阱。
>
> **规范出处**：Vol1 §A17.4（p.1936–1940）；§A17.5（p.1940–1942）；§A17.6（p.1942–1944）；§A17.8（p.1945–1946）；§A16.8–§A16.9（p.1920–1926）；§11.2.1.2（p.588–594）；§11.2.2（p.598–602）；§11.2.5.2（p.611–618）；§11.2.6（p.622–624）；§11.3（p.661–663）；§11.4（p.663–683）；§11.6.3（p.690–695）
>
> **前置阅读**：[01-resource-model.md](01-resource-model.md)、[02-qp-state-machine.md](02-qp-state-machine.md)、[03-memory-management.md](03-memory-management.md)、[04-work-requests-and-completions.md](04-work-requests-and-completions.md)
>
> **关键词**：CA17-30, Address Vector, GID type, SGID index, SL, Invalid Address Vector, Get Special QP, QP1, Raw, Client Reregistration, Port Change Event, libibverbs, ibv_modify_qp, ibv_query_gid_ex, rdma_cm

## 1. 总览

- **CA17-30**：RoCEv2 HCA 必须采纳 §A17.5 中描述的全部 Verbs 修改〔Vol1 §A17.5, p.1940〕。
- 修改分两类：① **地址向量（Address Vector, AV）相关**——凡是含 AV 的 Verb（AH、QP、XRC TGT QP、EE Context）都按 §A17.4.2 改写；② **删除只对 IB 链路层/子网管理有意义的内容**——LID/LMC、VL、SM 信息、MLID、QP0、Raw 服务、Client Reregistration、Port Change Event。
- RoCE v1（Annex A16）在 §A16.8–§A16.9 对同一批 Verb 做了几乎相同的修改；A17 在此基础上增加了端口类型值 “RoCEv2”、“RoCE Supported” 能力位以及**由 GID type 决定协议**的机制。

| Verb | RoCE v1（§A16.9） | RoCEv2（§A17.5） | 对编程的影响 |
|---|---|---|---|
| Query HCA | LID/LMC、VL 数、InitTypeReply、SM 信息、4 个 CapabilityMask 位无效；新增端口类型属性（RoCE 或 IB） | 同左；端口类型新增值 “RoCEv2”；新增 “RoCE Supported” 能力位 | 用 `link_layer` 与 GID 表项的 `gid_type` 判断；忽略 `lid`、`sm_lid` 等 |
| Modify HCA | IsSM、IsSNMPTunnelingSupported 位保留 | 同左 | — |
| Create/Modify/Query AH | DLID、Source Path Bits 不用；SGID index 与 DGID 对所有目的地都要设置；GRH 标志恒置位；保留 SL → Invalid Address Vector | AV 按 §A17.4.2 修改；使用保留 SL（8–15）可返回 Invalid Address Vector | `is_global = 1`，填写 `grh`，`sl ≤ 7` |
| Modify/Query QP、XRC INI/TGT QP | 主 AV 与备用路径同上 | 主 AV 与备用路径按 §A17.4.2 修改；保留 SL 可返回 Invalid Address Vector | 同上 |
| Modify/Query EE Context | 同上 | 同上 | 无（业界未实现 RD） |
| Attach/Detach 多播 | MLID 不用；删除 “Invalid multicast MLID” 结果 | 同左 | 只用 MGID |
| Poll for Completion | 数据报远端信息不含 SLID 与 DLID Path Bits | 数据报远端信息改为源 L2 地址 + IPv4/IPv6/RoCE 标志 | 用户态借助 `ibv_create_ah_from_wc()` |
| Get Special QP | 不再适用于 QP0 与 Raw | 只适用于 QP1，其它类型返回 “Invalid Special QP Type” | 只有 GSI（QP1） |
| Post Send | 删除 Raw 服务行（CA16-19） | 删除 Raw 服务行 | 没有 IB Raw QP |
| 非关联异步事件 | 不支持 Client Reregistration（CA16-20）与 Port Change Event（CA16-21） | 同左（CA17-31、CA17-32） | 不要等待这两个事件 |

〔Vol1 §A16.9, p.1921–1926；§A17.5, p.1940–1942〕

## 2. 逐条说明

### 2.1 Query HCA 与 Modify HCA

§A17.5.1 规定 RoCEv2 端口的端口属性列表变化如下〔Vol1 §A17.5.1, p.1940–1941〕：

- Base LID 与 LMC 不使用、应忽略；HCA 支持的最大 VL 数不使用；可选的 InitTypeReply 不使用；各端口的 Subnet Manager 地址信息不使用；
- CapabilityMask 中 IsSM、IsSMDisabled、IsSNMPTunnelingSupported、IsClientReregistrationSupported 不使用；
- **必须**为端口类型属性增加新值 “RoCEv2”；
- **必须**新增 “RoCE Supported” 能力位：只对 RoCEv2 类型端口有意义，置位表示该端口能**同时**工作在 RoCEv2 与 RoCE（v1）两种模式（见 §A17.8）。

§A17.5.2：Modify HCA 输入中 IsSM、IsSNMPTunnelingSupported 位不使用、作为保留〔Vol1 §A17.5.2, p.1941〕。

编程上：libibverbs 没有名为 “RoCEv2” 的端口类型字段，RoCE 端口表现为 `ibv_port_attr.link_layer == IBV_LINK_LAYER_ETHERNET`，协议版本体现在每个 GID 表项的类型上（`IBV_GID_TYPE_ROCE_V1` / `IBV_GID_TYPE_ROCE_V2`）〔业界实践: rdma-core〕。列出 GID 表：

```c
/* 列出端口 GID 表及其类型（rdma-core v32 及以上） */
static void dump_gid_table(struct ibv_context *ctx, uint8_t port, int gid_tbl_len)
{
    for (int i = 0; i < gid_tbl_len; i++) {
        struct ibv_gid_entry e;
        char s[INET6_ADDRSTRLEN];

        if (ibv_query_gid_ex(ctx, port, (uint32_t)i, &e, 0))
            continue;                           /* 空表项 */
        inet_ntop(AF_INET6, e.gid.raw, s, sizeof(s));
        printf("%3d %-7s %-40s ifindex=%u\n", i,
               e.gid_type == IBV_GID_TYPE_ROCE_V2 ? "RoCEv2" :
               e.gid_type == IBV_GID_TYPE_ROCE_V1 ? "RoCEv1" : "IB",
               s, e.ndev_ifindex);
    }
}
```

较老的 rdma-core 没有 `ibv_query_gid_ex()`，可读取 sysfs：`/sys/class/infiniband/<dev>/ports/<port>/gids/<index>` 与 `.../gid_attrs/types/<index>`（内容如 “RoCE v2”）〔业界实践: Linux，以内核版本为准〕。

### 2.2 地址向量的统一修改

所有含 AV 的 Verb 都按 §A17.4.2 修改〔Vol1 §A17.4.2, p.1937–1938；§A16.9.3–§A16.9.9, p.1921–1924〕：

| AV 分量 | IB 中的含义 | RoCE v1 / RoCEv2 中的处理 |
|---|---|---|
| Send GRH Flag | 跨子网或多播时置位 | **恒为 TRUE** |
| Source GID Index | 仅全局目的地需要 | 对**所有**目的地都要设置；所选表项的 GID type 决定出站协议（RoCE v1 / RoCEv2-IPv4 / RoCEv2-IPv6） |
| Destination GID | 仅全局目的地需要 | 对所有目的地都要设置；IPv4 写成 `::ffff:a.b.c.d` |
| Destination LID | 必需 | v1：不用（保留）；v2：泛化为 **Destination L2 Address**，可携带已解析的 DMAC；取 0xFFFFFFFFFFFF 表示由 Create AH / Modify QP 的实现自行做 L3→L2 解析 |
| Source Path Bits | 与 LMC 组成 SLID | 不适用 |
| Service Level | 映射到 VL | SL 0–7 直接映射为以太网优先级 0–7；SL 8–15 保留 |
| Flow Label / Hop Limit / Traffic Class | 写入 GRH | 写入 IP 头（Traffic Class → DSCP，Hop Limit → TTL/Hop Limit，Flow Label → IPv6 Flow Label） |
| 源 L2 地址 | 由端口 LID 决定 | 由所选 SGID 隐含，由实现借助 OS 服务取得 |

两个值得注意的差异：

1. **RoCE v1 禁止 AV 携带 L2 信息**（CA16-17：AV 中的源/目的标识只能是 GID，不得含 LID、Path Bits 等二层引用）〔Vol1 §A16.8, p.1920〕；**RoCEv2 则允许**通过 “Destination L2 Address” 传入已解析好的 DMAC——规范脚注说明这是为了适配 Linux RDMA CM 在生成 AV 之前就完成地址解析的做法〔Vol1 §A17.4.2, p.1937〕。〔业界实践: Linux kernel〕用户态的 `struct ibv_ah_attr` 没有 DMAC 字段，内核在处理用户态的 Create AH / Modify QP 时调用 `ib_resolve_eth_dmac()` 解析（相当于 0xFFFFFFFFFFFF 的语义）；内核态 ULP（如 rdma_cm）则直接提供已解析的 DMAC。
2. **保留 SL 的措辞**：RoCE v1 中使用保留 SL “shall” 得到 Invalid Address Vector（CA16-18）；A17 的 §A17.5.3–§A17.5.5 写的是 “may be returned”。可移植的程序应始终只用 SL 0–7。

### 2.3 Create / Modify / Query Address Handle

AV 按 2.2 节修改；Create/Modify AH 时，若 QP 关联 RoCEv2 端口而使用了保留 SL（8–15），可能返回 Invalid Address Vector〔Vol1 §A17.5.3, p.1941〕。AH 只用于 UD；Create AH 的其余输入（PD、静态速率、物理端口）不变〔Vol1 §11.2.2.1, p.598–599〕。

### 2.4 Modify / Query QP、XRC Target QP 与 EE Context

主 AV 与**备用路径（Alternate Path）**分量都按 §A17.4.2 修改；Modify QP / Modify XRC TGT QP / Modify EEC 使用保留 SL 时可能返回 Invalid Address Vector〔Vol1 §A17.5.4–§A17.5.5, p.1941〕。Modify QP 各属性在 RoCEv2 下的含义、`attr_mask` 对照与 APM 的处境见 [02-qp-state-machine.md](02-qp-state-machine.md)。

### 2.5 多播：Attach / Detach QP to/from Multicast Group

QP 关联 RoCEv2 端口时，多播组的 MLID 输入不使用、应忽略；“Invalid multicast MLID” 不再是有效的 Verb 结果〔Vol1 §A17.5.6, p.1942〕。因此 RoCEv2 多播组只由 **MGID**（IP 多播地址：IPv6 `ff00::/8`，或 IPv4 多播 `224.0.0.0/4` 的映射形式）标识；UD 多播发送的远端 QPN 用 0xFFFFFF（Table 100 脚注 c）〔Vol1 §11.4.1.1, p.668〕。

〔业界实践: Linux，以版本为准〕`ibv_attach_mcast(qp, &mgid, lid)` 的 `lid` 在 RoCE 上被忽略；内核把多播 GID 映射为以太网多播 MAC（`ib_resolve_eth_dmac()` 中的 `ip_eth_mc_map()`/`ipv6_eth_mc_map()`）；网络侧的 IGMP/MLD 成员关系通常交给 rdma_cm 的 `rdma_join_multicast()` 处理。多播语义见 [UC/UD 与多播](../02-transport/04-unreliable-services.md)。

### 2.6 Poll for Completion

数据报服务返回的远端端口地址与 QP 信息按 §A17.4.5.1 修改：远端地址信息由**源 L2 地址**和一个指示收到的是 **IPv4、IPv6 还是 RoCE** 报文的标志组成〔Vol1 §A17.5.7, p.1942；§A17.4.5.1, p.1939〕。用户态如何获得这些信息、UD 接收缓冲中 40 字节 L3 头区的布局（IPv4 头位于偏移 20），见 [04-work-requests-and-completions.md](04-work-requests-and-completions.md) 第 6–7 节。

### 2.7 Get Special QP

RoCEv2 端口没有 QP0，也不支持任何一种 Raw 数据报，因此 Get Special QP 只适用于 **GSI QP（QP1）**：C11-13 中关于 QP0 的要求与 o11-1 不适用；对 QP1 以外的类型调用必须返回 **“Invalid Special QP Type”**〔Vol1 §A17.5.8, p.1942〕。目的 QP 为 0 的入站报文被静默丢弃（CA17-33）〔Vol1 §A17.6, p.1943〕。CM 报文就是 QP1 上与目标连接同类型（IPv4/IPv6）的普通 RoCEv2 报文（§A17.6.1），详见 [MAD 与 GSI](../04-connection-management/03-mad-and-gsi.md)。

〔业界实践: Linux〕QP1 由内核 MAD 层独占，用户态程序不能直接创建 QP1，CM 交互通过 rdma_cm（或 `ib_umad` 等接口）完成。

### 2.8 Post Send Request

删除 Raw 服务类型：Post Send 的 “Operation Type Matrix”（Table 99）中 Raw 一行被删去〔Vol1 §A17.5.9, p.1942〕；与之对应，RoCEv2 端口不得支持 IB Raw 数据报（CA17-28）〔Vol1 §A17.4.6, p.1939〕。libibverbs 的 `IBV_QPT_RAW_PACKET` 是厂商扩展的原始以太网 QP，不是 IBA Raw 数据报〔业界实践〕。

### 2.9 异步事件

CA17-31：RoCEv2 端口不得支持 Client Reregistration；CA17-32：不得支持可选的 Port Change Event〔Vol1 §A17.5.10, p.1942〕。§11.6.3 的完整事件表与 libibverbs 对应如下〔Vol1 §11.6.3, p.690–695；业界实践: rdma-core `enum ibv_event_type`〕：

| 类别 | 规范事件 | `ibv_event_type` | 对 QP 状态 | RoCEv2 备注 |
|---|---|---|---|---|
| 关联事件 | Path Migrated | `IBV_EVENT_PATH_MIG` | 不变（C11-34） | Linux 在 RoCE 上禁用 APM |
| 关联事件 | Communication Established | `IBV_EVENT_COMM_EST` | 不变 | RTR 下首包到达（C11-35） |
| 关联事件 | Send Queue Drained | `IBV_EVENT_SQ_DRAINED` | 不变 | |
| 关联事件 | SRQ Limit Reached | `IBV_EVENT_SRQ_LIMIT_REACHED` | 不变 | |
| 关联事件 | Last WQE Reached | `IBV_EVENT_QP_LAST_WQE_REACHED` | 不变 | 关联 SRQ 的 QP 拆除流程用 |
| 关联错误 | CQ Error | `IBV_EVENT_CQ_ERR` | 关联 QP → Error | 溢出或不可访问 |
| 关联错误 | Local Work Queue Catastrophic Error | `IBV_EVENT_QP_FATAL` | → Error（C11-33） | |
| 关联错误 | Invalid Request Local WQ Error | `IBV_EVENT_QP_REQ_ERR` | → Error | 响应方检测到 OpCode 违例 |
| 关联错误 | Local Access Violation WQ Error | `IBV_EVENT_QP_ACCESS_ERR` | → Error | R_Key 违例、未对齐 Atomic、READ/Atomic 过多等 |
| 关联错误 | Path Migration Request Error | `IBV_EVENT_PATH_MIG_ERR` | → Error | |
| 关联错误 | SRQ Catastrophic Error | `IBV_EVENT_SRQ_ERR` | SRQ → Error | |
| 非关联事件 | Port Active | `IBV_EVENT_PORT_ACTIVE` | — | 链路由不可用变为可用 |
| 非关联事件 | Client Reregistration | `IBV_EVENT_CLIENT_REREGISTER` | — | **不得支持**（CA17-31） |
| 非关联事件 | Port Change（带变化标志） | Linux 拆成 `IBV_EVENT_LID_CHANGE`、`IBV_EVENT_PKEY_CHANGE`、`IBV_EVENT_SM_CHANGE`、`IBV_EVENT_GID_CHANGE` 等 | — | **不得支持** Port Change（CA17-32）；Linux 另用 `IBV_EVENT_GID_CHANGE` 通知 GID 表变化（非 IBA 定义） |
| 非关联错误 | Local Catastrophic Error | `IBV_EVENT_DEVICE_FATAL` | 不确定 | |
| 非关联错误 | Port Error | `IBV_EVENT_PORT_ERR` | **不影响** QP 状态（C11-41） | 链路由可用变为不可用 |

异步事件需用 `ibv_get_async_event()` 读取并以 `ibv_ack_async_event()` 确认；注册处理器之前发生的异步错误会丢失（C10-138）。

### 2.10 其它影响编程的 A17 规定

- **与 RoCE v1 互通**（可选，§A17.8）：支持 RoCE 的 RoCEv2 端口上，连接型 QP 按 Modify QP 时所配 GID 表项的 GID type 决定工作在 RoCE 还是 RoCEv2 模式（CA17-40）；入站报文做 DGID 检查时，若所比对表项的 GID type 与报文协议不符，检查视为失败（CA17-41）；UD QP 按每个 WQE 所引用 AV 的 GID type 选择模式（CA17-42），并且两种报文都接收（CA17-43）〔Vol1 §A17.8, p.1945–1946〕。**两端必须选同一类型的 GID**，否则报文被对端静默丢弃。
- **P_Key**：表的填充方式实现自定，至少有默认 P_Key；BTH.P_Key 照常校验（§A17.4.7），见 [01-resource-model.md](01-resource-model.md)。
- **管理类**：SM、SA、PM 等管理类不适用，RoCEv2 端口接入由常规以太网/IP 手段管理的 IP 子网（§A17.6）——程序不能依赖 SA PathRecord 获取路径参数，PMTU、超时等要由应用或 rdma_cm 给出。

## 3. 完整示例：libibverbs RC over RoCEv2

### 3.1 程序结构与规范对应

| 步骤 | libibverbs 调用〔业界实践〕 | 对应 Verb / 规范 |
|---|---|---|
| 打开设备 | `ibv_get_device_list()`、`ibv_open_device()` | Open HCA（§11.2.1.1） |
| 查询能力与端口 | `ibv_query_device()`、`ibv_query_port()` | Query HCA（§11.2.1.2、§A17.5.1） |
| 选 RoCEv2 GID | `ibv_query_gid_ex()`、`ibv_query_gid()` | GID 表与 GID type（CA17-26、§A17.4.3） |
| 创建 PD / CQ | `ibv_alloc_pd()`、`ibv_create_cq()` | §11.2.1.5、§11.2.8.1 |
| 注册内存 | `ibv_reg_mr()` | Register MR（§11.2.10.2） |
| 创建 QP | `ibv_create_qp()` | Create QP（§11.2.5.1） |
| RESET→INIT→RTR→RTS | `ibv_modify_qp()` | Modify QP（§11.2.5.2 Table 96、§A17.5.4） |
| 投递 | `ibv_post_recv()`、`ibv_post_send()` | §11.4.1 |
| 取完成 | `ibv_poll_cq()` | Poll for Completion（§11.4.2.1） |
| 清理 | `ibv_destroy_qp()` → `ibv_dereg_mr()` → `ibv_destroy_cq()` → `ibv_dealloc_pd()` → `ibv_close_device()` | §11.2.5.4、§11.2.10.5、§11.2.8.4、§11.2.1.6、§11.2.1.4 |

通信流程：两端各自建好资源并把 QP 置于 INIT、预投递一个接收 WR；通过 TCP 交换 QPN、起始 PSN、GID、rkey、缓冲地址与 MTU；各自迁移到 RTR、RTS；客户端在同一 SQ 上先投递一个 **unsignaled 的 RDMA WRITE**，再投递一个 **signaled 的 SEND with Immediate**；服务端收到 SEND 的完成后即可读取 RDMA WRITE 写入的数据（RC 请求按序执行），客户端通过 SEND 的完成确认之前的 WRITE 也已完成（§10.8.6）。

### 3.2 源码

```c
/*
 * rc_roce_demo.c —— RoCEv2 上 RC QP 的最小完整示例（libibverbs + TCP 带外交换）
 *
 * 编译：  gcc -O2 -Wall -o rc_roce_demo rc_roce_demo.c -libverbs
 * 服务端：./rc_roce_demo [-d mlx5_0] [-i 1] [-g 3] [-p 18515]
 * 客户端：./rc_roce_demo [-d mlx5_0] [-i 1] [-g 3] [-p 18515] <服务端 IP>
 *
 * 流程：打开设备 → 查询端口与 GID 表并选 RoCEv2 GID → 创建 PD/CQ/MR/QP
 *      → RESET→INIT → 预投递接收 → TCP 交换 QPN/PSN/GID/rkey/addr/MTU
 *      → INIT→RTR→RTS → 客户端 RDMA WRITE + SEND with IMM，服务端收到 SEND 后
 *        读出 RDMA WRITE 写入的数据 → 清理
 * 需要 rdma-core v32 及以上（ibv_query_gid_ex）。
 */
#define _GNU_SOURCE
#include <arpa/inet.h>
#include <endian.h>
#include <errno.h>
#include <infiniband/verbs.h>
#include <netdb.h>
#include <netinet/in.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <time.h>
#include <unistd.h>

#define BUF_SIZE     4096
#define WRITE_OFF    0          /* RDMA WRITE 写入对端缓冲的位置 */
#define SEND_SRC_OFF 1024       /* 本端 SEND 的源数据位置 */
#define RECV_OFF     2048       /* 本端接收 SEND 的位置 */
#define QUEUE_DEPTH  16
#define WRID_WRITE   1
#define WRID_SEND    2
#define WRID_RECV    3

/* 带外交换的连接参数；线上一律使用网络字节序 */
struct conn_info {
    uint32_t qpn;
    uint32_t psn;
    uint32_t rkey;
    uint32_t mtu;               /* enum ibv_mtu */
    uint64_t addr;
    uint8_t  gid[16];
};

struct ctx {
    struct ibv_context *ctx;
    struct ibv_pd      *pd;
    struct ibv_cq      *cq;
    struct ibv_qp      *qp;
    struct ibv_mr      *mr;
    char               *buf;
    uint8_t             port;
    int                 gid_idx;
    union ibv_gid       gid;
    enum ibv_mtu        mtu;
};

static void die(const char *what, int err)
{
    fprintf(stderr, "%s failed: %s\n", what, strerror(err));
    exit(EXIT_FAILURE);
}

/* ---------- 1. GID 选择：只接受 RoCEv2 类型，优先 IPv4 映射地址 ---------- */
static int gid_score(const struct ibv_gid_entry *e)
{
    static const uint8_t v4mapped[12] = { 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0xff, 0xff };
    const uint8_t *g = e->gid.raw;

    if (e->gid_type != IBV_GID_TYPE_ROCE_V2)
        return -1;                              /* RoCE v1 / IB 类型的表项不要 */
    if (memcmp(g, v4mapped, sizeof(v4mapped)) == 0)
        return 3;                               /* ::ffff:a.b.c.d */
    if (g[0] == 0xfe && (g[1] & 0xc0) == 0x80)
        return 1;                               /* fe80::/10 只能在同一二层内通信 */
    return 2;                                   /* 全局 IPv6 */
}

static int pick_rocev2_gid(struct ibv_context *ctx, uint8_t port, int tbl_len)
{
    int best = -1, best_score = -1;

    for (int i = 0; i < tbl_len; i++) {
        struct ibv_gid_entry e;

        if (ibv_query_gid_ex(ctx, port, (uint32_t)i, &e, 0))
            continue;                           /* ENODATA：空表项 */
        int s = gid_score(&e);
        if (s > best_score) {
            best_score = s;
            best = i;
        }
    }
    return best;
}

/* ---------- 2. TCP 带外通道 ---------- */
static int tcp_connect(const char *host, int port)
{
    struct addrinfo hints, *res, *rp;
    char service[16];
    int fd = -1;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    snprintf(service, sizeof(service), "%d", port);
    if (getaddrinfo(host, service, &hints, &res))
        return -1;
    for (rp = res; rp; rp = rp->ai_next) {
        fd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
        if (fd < 0)
            continue;
        if (connect(fd, rp->ai_addr, rp->ai_addrlen) == 0)
            break;
        close(fd);
        fd = -1;
    }
    freeaddrinfo(res);
    return fd;
}

static int tcp_accept_one(int port)
{
    struct sockaddr_in sa;
    int one = 1, lfd, fd;

    lfd = socket(AF_INET, SOCK_STREAM, 0);
    if (lfd < 0)
        return -1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, (const void *)&one, sizeof(one));
    memset(&sa, 0, sizeof(sa));
    sa.sin_family = AF_INET;
    sa.sin_addr.s_addr = htonl(INADDR_ANY);
    sa.sin_port = htons((uint16_t)port);
    if (bind(lfd, (struct sockaddr *)&sa, sizeof(sa)) || listen(lfd, 1)) {
        close(lfd);
        return -1;
    }
    fd = accept(lfd, NULL, NULL);
    close(lfd);
    return fd;
}

static void write_all(int fd, const void *p, size_t n)
{
    const char *c = p;

    while (n) {
        ssize_t r = write(fd, c, n);
        if (r <= 0)
            die("write", errno);
        c += r;
        n -= (size_t)r;
    }
}

static void read_all(int fd, void *p, size_t n)
{
    char *c = p;

    while (n) {
        ssize_t r = read(fd, c, n);
        if (r <= 0)
            die("read", r == 0 ? ECONNRESET : errno);
        c += r;
        n -= (size_t)r;
    }
}

static void exchange(int fd, const struct conn_info *loc, struct conn_info *rem)
{
    struct conn_info out, in;

    out.qpn  = htonl(loc->qpn);
    out.psn  = htonl(loc->psn);
    out.rkey = htonl(loc->rkey);
    out.mtu  = htonl(loc->mtu);
    out.addr = htobe64(loc->addr);
    memcpy(out.gid, loc->gid, sizeof(out.gid));
    write_all(fd, &out, sizeof(out));           /* 先写后读：40 字节不会阻塞 */
    read_all(fd, &in, sizeof(in));
    rem->qpn  = ntohl(in.qpn);
    rem->psn  = ntohl(in.psn);
    rem->rkey = ntohl(in.rkey);
    rem->mtu  = ntohl(in.mtu);
    rem->addr = be64toh(in.addr);
    memcpy(rem->gid, in.gid, sizeof(rem->gid));
}

static void barrier(int fd)                     /* 简单的双向同步点 */
{
    char ch = 'x';

    write_all(fd, &ch, 1);
    read_all(fd, &ch, 1);
}

/* ---------- 3. QP 状态迁移（属性与 attr_mask 对照 Table 96） ---------- */
static int qp_to_init(struct ctx *c)
{
    struct ibv_qp_attr a;

    memset(&a, 0, sizeof(a));
    a.qp_state        = IBV_QPS_INIT;
    a.pkey_index      = 0;                      /* RoCE 的 P_Key 表通常只有默认 0xFFFF */
    a.port_num        = c->port;
    a.qp_access_flags = IBV_ACCESS_REMOTE_WRITE | IBV_ACCESS_REMOTE_READ;
    return ibv_modify_qp(c->qp, &a, IBV_QP_STATE | IBV_QP_PKEY_INDEX |
                                    IBV_QP_PORT | IBV_QP_ACCESS_FLAGS);
}

static int qp_to_rtr(struct ctx *c, const struct conn_info *r)
{
    struct ibv_qp_attr a;

    memset(&a, 0, sizeof(a));
    a.qp_state           = IBV_QPS_RTR;
    a.path_mtu           = c->mtu;              /* ≤ 两端 active_mtu */
    a.dest_qp_num        = r->qpn;
    a.rq_psn             = r->psn;              /* = 对端的 SQ PSN */
    a.max_dest_rd_atomic = 1;                   /* Responder Resources */
    a.min_rnr_timer      = 12;                  /* 编码 12 = 0.64 ms（Table 48） */
    a.ah_attr.is_global  = 1;                   /* RoCE：GRH 标志恒为 TRUE */
    a.ah_attr.port_num   = c->port;
    a.ah_attr.sl         = 0;                   /* → 以太网优先级 0；8–15 保留 */
    a.ah_attr.dlid       = 0;                   /* RoCE 不使用 LID */
    a.ah_attr.src_path_bits = 0;
    memcpy(a.ah_attr.grh.dgid.raw, r->gid, 16); /* 目的 IP（IPv4 为 ::ffff:x.x.x.x） */
    a.ah_attr.grh.sgid_index    = (uint8_t)c->gid_idx;  /* 源 IP 与协议类型 */
    a.ah_attr.grh.hop_limit     = 64;           /* IPv4 TTL / IPv6 Hop Limit */
    a.ah_attr.grh.traffic_class = 0;            /* Linux 中为整个 TOS 字节 */
    a.ah_attr.grh.flow_label    = 0;            /* 0：驱动可按 QPN 生成 UDP 源端口 */
    return ibv_modify_qp(c->qp, &a, IBV_QP_STATE | IBV_QP_AV | IBV_QP_PATH_MTU |
                                    IBV_QP_DEST_QPN | IBV_QP_RQ_PSN |
                                    IBV_QP_MAX_DEST_RD_ATOMIC | IBV_QP_MIN_RNR_TIMER);
}

static int qp_to_rts(struct ctx *c, uint32_t my_psn)
{
    struct ibv_qp_attr a;

    memset(&a, 0, sizeof(a));
    a.qp_state      = IBV_QPS_RTS;
    a.timeout       = 14;                       /* Ttr = 4.096 us × 2^14 ≈ 67 ms */
    a.retry_cnt     = 7;                        /* 3 bit 最大值 */
    a.rnr_retry     = 7;                        /* 7 = 无限重试（演示用） */
    a.sq_psn        = my_psn;
    a.max_rd_atomic = 1;                        /* Initiator Depth ≤ 对端 Responder Resources */
    return ibv_modify_qp(c->qp, &a, IBV_QP_STATE | IBV_QP_TIMEOUT | IBV_QP_RETRY_CNT |
                                    IBV_QP_RNR_RETRY | IBV_QP_SQ_PSN |
                                    IBV_QP_MAX_QP_RD_ATOMIC);
}

/* ---------- 4. 投递与轮询 ---------- */
static int post_recv(struct ctx *c)
{
    struct ibv_sge sge;
    struct ibv_recv_wr wr, *bad = NULL;

    sge.addr   = (uintptr_t)(c->buf + RECV_OFF);
    sge.length = BUF_SIZE - RECV_OFF;
    sge.lkey   = c->mr->lkey;
    memset(&wr, 0, sizeof(wr));
    wr.wr_id   = WRID_RECV;
    wr.sg_list = &sge;
    wr.num_sge = 1;
    return ibv_post_recv(c->qp, &wr, &bad);
}

static int post_write_then_send(struct ctx *c, const struct conn_info *r)
{
    static const char wmsg[] = "hello from RDMA WRITE";
    static const char smsg[] = "hello from SEND";
    struct ibv_sge wsge, ssge;
    struct ibv_send_wr wwr, swr, *bad = NULL;

    memcpy(c->buf + WRITE_OFF, wmsg, sizeof(wmsg));
    memcpy(c->buf + SEND_SRC_OFF, smsg, sizeof(smsg));
    wsge.addr = (uintptr_t)(c->buf + WRITE_OFF);
    wsge.length = sizeof(wmsg);
    wsge.lkey = c->mr->lkey;
    ssge.addr = (uintptr_t)(c->buf + SEND_SRC_OFF);
    ssge.length = sizeof(smsg);
    ssge.lkey = c->mr->lkey;

    memset(&wwr, 0, sizeof(wwr));
    wwr.wr_id      = WRID_WRITE;
    wwr.opcode     = IBV_WR_RDMA_WRITE;
    wwr.sg_list    = &wsge;
    wwr.num_sge    = 1;
    wwr.send_flags = 0;                         /* unsignaled：由后面 SEND 的完成来确认 */
    wwr.wr.rdma.remote_addr = r->addr + WRITE_OFF;
    wwr.wr.rdma.rkey        = r->rkey;
    wwr.next       = &swr;                      /* 一次投递两个 WR */

    memset(&swr, 0, sizeof(swr));
    swr.wr_id      = WRID_SEND;
    swr.opcode     = IBV_WR_SEND_WITH_IMM;
    swr.imm_data   = htonl(0x12345678);         /* 立即数：网络字节序 */
    swr.sg_list    = &ssge;
    swr.num_sge    = 1;
    swr.send_flags = IBV_SEND_SIGNALED | IBV_SEND_SOLICITED;
    return ibv_post_send(c->qp, &wwr, &bad);
}

static int poll_one(struct ibv_cq *cq, struct ibv_wc *wc, int timeout_ms)
{
    struct timespec t0, t;

    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (;;) {
        int n = ibv_poll_cq(cq, 1, wc);
        if (n < 0)
            return -1;
        if (n == 1) {
            if (wc->status != IBV_WC_SUCCESS) {
                fprintf(stderr, "WC error: %s (%d), vendor_err 0x%x, wr_id %llu\n",
                        ibv_wc_status_str(wc->status), (int)wc->status,
                        wc->vendor_err, (unsigned long long)wc->wr_id);
                return -1;
            }
            return 0;
        }
        clock_gettime(CLOCK_MONOTONIC, &t);
        if ((t.tv_sec - t0.tv_sec) * 1000 + (t.tv_nsec - t0.tv_nsec) / 1000000 > timeout_ms) {
            fprintf(stderr, "poll CQ timeout\n");
            return -1;
        }
    }
}

int main(int argc, char **argv)
{
    const char *dev_name = NULL, *server_ip = NULL;
    int ib_port = 1, gid_idx = -1, tcp_port = 18515, opt, ret, num = 0, fd;
    struct ibv_device **list;
    struct ibv_device *dev = NULL;
    struct ibv_device_attr dattr;
    struct ibv_port_attr pattr;
    struct ibv_qp_init_attr qia;
    struct conn_info loc, rem;
    struct ibv_wc wc;
    char gstr[INET6_ADDRSTRLEN];
    struct ctx c;

    while ((opt = getopt(argc, argv, "d:i:g:p:")) != -1) {
        switch (opt) {
        case 'd': dev_name = optarg; break;
        case 'i': ib_port = atoi(optarg); break;
        case 'g': gid_idx = atoi(optarg); break;
        case 'p': tcp_port = atoi(optarg); break;
        default:
            fprintf(stderr, "usage: %s [-d dev] [-i port] [-g gid_idx] [-p tcp_port] [server_ip]\n",
                    argv[0]);
            return EXIT_FAILURE;
        }
    }
    if (optind < argc)
        server_ip = argv[optind];               /* 给出对端地址即为客户端 */
    memset(&c, 0, sizeof(c));
    c.port = (uint8_t)ib_port;

    /* 1. 打开设备 */
    list = ibv_get_device_list(&num);
    if (!list || num == 0)
        die("ibv_get_device_list", ENODEV);
    for (int i = 0; i < num; i++) {
        if (!dev_name || strcmp(ibv_get_device_name(list[i]), dev_name) == 0) {
            dev = list[i];
            break;
        }
    }
    if (!dev)
        die("find device", ENODEV);
    c.ctx = ibv_open_device(dev);
    if (!c.ctx)
        die("ibv_open_device", errno);

    /* 2. 查询设备与端口（Query HCA） */
    if ((ret = ibv_query_device(c.ctx, &dattr)))
        die("ibv_query_device", ret);
    if ((ret = ibv_query_port(c.ctx, c.port, &pattr)))
        die("ibv_query_port", ret);
    if (pattr.link_layer != IBV_LINK_LAYER_ETHERNET)
        die("port check (link_layer is not Ethernet, not a RoCE port)", EINVAL);
    if (pattr.state != IBV_PORT_ACTIVE)
        die("port check (port is not ACTIVE)", ENETDOWN);
    printf("%s: max_qp=%d max_qp_wr=%d max_qp_rd_atom=%d active_mtu=%d\n",
           ibv_get_device_name(dev), dattr.max_qp, dattr.max_qp_wr,
           dattr.max_qp_rd_atom, 128 << pattr.active_mtu);

    /* 3. 选择 RoCEv2 GID（-g 可强制指定） */
    c.gid_idx = gid_idx >= 0 ? gid_idx : pick_rocev2_gid(c.ctx, c.port, pattr.gid_tbl_len);
    if (c.gid_idx < 0)
        die("pick RoCEv2 GID", ENOENT);
    if (ibv_query_gid(c.ctx, c.port, c.gid_idx, &c.gid))
        die("ibv_query_gid", errno ? errno : EINVAL);
    inet_ntop(AF_INET6, c.gid.raw, gstr, sizeof(gstr));
    printf("using gid_index %d: %s\n", c.gid_idx, gstr);
    c.mtu = pattr.active_mtu;

    /* 4. 创建 PD / CQ / MR / QP */
    c.pd = ibv_alloc_pd(c.ctx);
    if (!c.pd)
        die("ibv_alloc_pd", errno);
    c.cq = ibv_create_cq(c.ctx, 2 * QUEUE_DEPTH, NULL, NULL, 0);
    if (!c.cq)
        die("ibv_create_cq", errno);
    if ((ret = posix_memalign((void **)&c.buf, 4096, BUF_SIZE)))
        die("posix_memalign", ret);
    memset(c.buf, 0, BUF_SIZE);
    c.mr = ibv_reg_mr(c.pd, c.buf, BUF_SIZE, IBV_ACCESS_LOCAL_WRITE |
                      IBV_ACCESS_REMOTE_WRITE | IBV_ACCESS_REMOTE_READ);
    if (!c.mr)
        die("ibv_reg_mr", errno);
    memset(&qia, 0, sizeof(qia));
    qia.send_cq          = c.cq;
    qia.recv_cq          = c.cq;
    qia.qp_type          = IBV_QPT_RC;
    qia.sq_sig_all       = 0;                   /* Selectable：由每个 WR 决定是否产生 CQE */
    qia.cap.max_send_wr  = QUEUE_DEPTH;
    qia.cap.max_recv_wr  = QUEUE_DEPTH;
    qia.cap.max_send_sge = 1;
    qia.cap.max_recv_sge = 1;
    c.qp = ibv_create_qp(c.pd, &qia);
    if (!c.qp)
        die("ibv_create_qp", errno);

    /* 5. RESET → INIT，并在对端可能发包之前预投递接收 */
    if ((ret = qp_to_init(&c)))
        die("modify QP to INIT", ret);
    if ((ret = post_recv(&c)))
        die("ibv_post_recv", ret);

    /* 6. 带外交换 QPN / PSN / GID / rkey / addr / MTU */
    fd = server_ip ? tcp_connect(server_ip, tcp_port) : tcp_accept_one(tcp_port);
    if (fd < 0)
        die("TCP connect/accept", errno ? errno : ECONNREFUSED);
    srand48((long)getpid() ^ (long)time(NULL));
    memset(&loc, 0, sizeof(loc));
    loc.qpn  = c.qp->qp_num;
    loc.psn  = (uint32_t)lrand48() & 0xFFFFFF;  /* 24 bit 随机起始 PSN */
    loc.rkey = c.mr->rkey;
    loc.addr = (uint64_t)(uintptr_t)c.buf;
    loc.mtu  = (uint32_t)c.mtu;
    memcpy(loc.gid, c.gid.raw, 16);
    exchange(fd, &loc, &rem);
    if (rem.mtu < (uint32_t)c.mtu)
        c.mtu = (enum ibv_mtu)rem.mtu;          /* 两端取较小的 path MTU */
    printf("local QPN 0x%06x PSN 0x%06x | remote QPN 0x%06x PSN 0x%06x rkey 0x%08x\n",
           loc.qpn, loc.psn, rem.qpn, rem.psn, rem.rkey);

    /* 7. INIT → RTR → RTS */
    if ((ret = qp_to_rtr(&c, &rem)))
        die("modify QP to RTR", ret);
    if ((ret = qp_to_rts(&c, loc.psn)))
        die("modify QP to RTS", ret);
    barrier(fd);                                /* 双方都已进入 RTS 且已预投递接收 */

    /* 8. 数据传输 */
    if (server_ip) {                            /* 客户端：RDMA WRITE + SEND with IMM */
        if ((ret = post_write_then_send(&c, &rem)))
            die("ibv_post_send", ret);
        if (poll_one(c.cq, &wc, 5000) || wc.wr_id != WRID_SEND)
            die("wait for SEND completion", EIO);
        printf("client: RDMA WRITE + SEND completed\n");
    } else {                                    /* 服务端：等待 SEND 的接收完成 */
        if (poll_one(c.cq, &wc, 30000) || wc.wr_id != WRID_RECV)
            die("wait for RECV completion", EIO);
        printf("server: recv %u bytes, imm 0x%08x: \"%s\"\n", wc.byte_len,
               (wc.wc_flags & IBV_WC_WITH_IMM) ? (unsigned)ntohl(wc.imm_data) : 0u,
               c.buf + RECV_OFF);
        /* RC 按序执行：SEND 的完成到达时，之前的 RDMA WRITE 数据已写入 */
        printf("server: RDMA WRITE data: \"%s\"\n", c.buf + WRITE_OFF);
    }
    barrier(fd);
    close(fd);

    /* 9. 清理：QP → MR → CQ → PD → 设备 */
    ibv_destroy_qp(c.qp);
    ibv_dereg_mr(c.mr);
    free(c.buf);
    ibv_destroy_cq(c.cq);
    ibv_dealloc_pd(c.pd);
    ibv_close_device(c.ctx);
    ibv_free_device_list(list);
    return 0;
}
```

### 3.3 编译与运行

```bash
# 需要 rdma-core（libibverbs 开发包），v32 及以上提供 ibv_query_gid_ex()
gcc -O2 -Wall -o rc_roce_demo rc_roce_demo.c -libverbs

# 服务端（假设 RoCE 网卡 mlx5_0，端口 1；不指定 -g 时自动挑选 RoCEv2 GID）
./rc_roce_demo -d mlx5_0 -i 1

# 客户端
./rc_roce_demo -d mlx5_0 -i 1 192.168.10.1
```

服务端输出形如（数值仅为示意，因设备与配置而异）：

```text
mlx5_0: max_qp=... max_qp_wr=... max_qp_rd_atom=... active_mtu=4096
using gid_index 3: ::ffff:192.168.10.1
local QPN 0x0001a4 PSN 0x3c91e2 | remote QPN 0x0001b7 PSN 0x8f0a13 rkey 0x...
server: recv 16 bytes, imm 0x12345678: "hello from SEND"
server: RDMA WRITE data: "hello from RDMA WRITE"
```

> [!NOTE]
> 该程序已用 rdma-core 公开头文件做过编译检查（`-Wall -Wextra` 无告警），未在真实 RNIC 上运行；不同驱动对个别参数（如 `max_dest_rd_atomic` 的上取整、`timeout` 的最小值）可能有自己的限制，以设备文档为准。没有 RoCE 网卡时，可在 Linux 上用内核自带的 Soft-RoCE（rxe）驱动做功能验证〔业界实践: Linux rdma_rxe〕。

### 3.4 关键点解读

1. **只选 RoCEv2 类型的 GID**：`gid_score()` 排除 RoCE v1/IB 表项，并优先 IPv4 映射地址、其次全局 IPv6、最后链路本地地址（`fe80::/10` 只能在同一二层内通信）。两端必须选同一类型（CA17-40、CA17-41）。生产环境中通常应选与“去往对端的路由出口网卡/VLAN”对应的那一项，`-g` 允许手工指定。
2. **AV 必填项**：`ah_attr.is_global = 1`、`grh.sgid_index`、`grh.dgid`、`grh.hop_limit`；`sl` 只用 0–7；`dlid` 与 `src_path_bits` 在 RoCE 中无意义。DMAC 由内核解析，`hop_limit` 也可能被内核按路由结果覆盖（见 [02-qp-state-machine.md](02-qp-state-machine.md) 第 6 节）。
3. **path MTU 取两端较小值**：程序交换各自的 `active_mtu` 并取较小者，避免“对端 PMTU 更小导致长度校验失败”。
4. **PSN**：24 bit 随机值，本端 `sq_psn` = 自己交换出去的 PSN，本端 `rq_psn` = 对端交换来的 PSN。
5. **INIT 时就预投递接收**：规范允许 INIT 下投递接收（C10-24）。这样 QP 一进入 RTR 就有可用的接收 WQE；若本端已处于 RTR/RTS 而 RQ 为空时对端发来 SEND，RC 会回 RNR NAK，请求方要等待 `min_rnr_timer` 指示的时间再重试（本例另用 TCP 同步点保证双方都进入 RTS 后才开始发送）。
6. **unsignaled + signaled 组合**：WRITE 成功时不产生 CQE；取到后面 SEND 的 WC 即确认 WRITE 完成并可复用其缓冲（C10-108）。
7. **立即数用网络字节序**：发送端 `htonl()`，接收端 `ntohl()`；接收端 `byte_len` 不含立即数。
8. **RDMA READ 的本地缓冲需要 Local Write**：本例 MR 带 `IBV_ACCESS_LOCAL_WRITE`，既能作接收缓冲，也能作 READ 的落地缓冲。
9. **清理顺序**：先销毁 QP，再注销 MR、销毁 CQ、释放 PD（PD/CQ 仍被引用时会报错，C10-11.2.1、C10-18）。

## 4. UD over RoCEv2 片段

```c
/* UD over RoCEv2 关键片段（需 #include <infiniband/verbs.h>、<string.h>） */

/* 1) UD QP 三步迁移：INIT 需 P_Key index、端口、Q_Key；RTR 只需状态；RTS 需 SQ PSN */
static int ud_qp_bringup(struct ibv_qp *qp, uint8_t port, uint32_t qkey, uint32_t psn)
{
    struct ibv_qp_attr a;

    memset(&a, 0, sizeof(a));
    a.qp_state   = IBV_QPS_INIT;
    a.pkey_index = 0;
    a.port_num   = port;
    a.qkey       = qkey;
    if (ibv_modify_qp(qp, &a, IBV_QP_STATE | IBV_QP_PKEY_INDEX | IBV_QP_PORT | IBV_QP_QKEY))
        return -1;
    a.qp_state = IBV_QPS_RTR;
    if (ibv_modify_qp(qp, &a, IBV_QP_STATE))
        return -1;
    a.qp_state = IBV_QPS_RTS;
    a.sq_psn   = psn & 0xFFFFFF;
    return ibv_modify_qp(qp, &a, IBV_QP_STATE | IBV_QP_SQ_PSN);
}

/* 2) 为目的端创建 AH：is_global 必须为 1，SGID 必须是 RoCEv2 类型的表项 */
static struct ibv_ah *make_rocev2_ah(struct ibv_pd *pd, uint8_t port, int sgid_idx,
                                     const union ibv_gid *dgid)
{
    struct ibv_ah_attr ah;

    memset(&ah, 0, sizeof(ah));
    ah.is_global      = 1;
    ah.port_num       = port;
    ah.sl             = 0;                     /* 0–7 → 以太网优先级 */
    ah.grh.dgid       = *dgid;                 /* IPv4 对端写成 ::ffff:a.b.c.d */
    ah.grh.sgid_index = (uint8_t)sgid_idx;
    ah.grh.hop_limit  = 64;
    return ibv_create_ah(pd, &ah);
}

/* 3) 发送：每个 WR 指定 AH、远端 QPN 与 Q_Key */
static int ud_send(struct ibv_qp *qp, struct ibv_ah *ah, uint32_t rqpn, uint32_t qkey,
                   void *buf, uint32_t len, uint32_t lkey)
{
    struct ibv_sge sge;
    struct ibv_send_wr wr, *bad = NULL;

    sge.addr   = (uintptr_t)buf;
    sge.length = len;                          /* ≤ path MTU，否则规范要求不发包也不报错 */
    sge.lkey   = lkey;
    memset(&wr, 0, sizeof(wr));
    wr.opcode            = IBV_WR_SEND;
    wr.sg_list           = &sge;
    wr.num_sge           = 1;
    wr.send_flags        = IBV_SEND_SIGNALED;
    wr.wr.ud.ah          = ah;
    wr.wr.ud.remote_qpn  = rqpn;
    wr.wr.ud.remote_qkey = qkey;               /* 最高位为 1 时改用本 QP 的 Q_Key（C10-15） */
    return ibv_post_send(qp, &wr, &bad);
}

/* 4) 接收缓冲 = 40 字节 L3 头区 + 负载区 */
static int ud_post_recv(struct ibv_qp *qp, char *buf, uint32_t mtu_bytes, uint32_t lkey)
{
    struct ibv_sge sge;
    struct ibv_recv_wr wr, *bad = NULL;

    sge.addr   = (uintptr_t)buf;
    sge.length = 40 + mtu_bytes;
    sge.lkey   = lkey;
    memset(&wr, 0, sizeof(wr));
    wr.sg_list = &sge;
    wr.num_sge = 1;
    return ibv_post_recv(qp, &wr, &bad);
}

/* 5) 收到后：负载位于 buf + 40，长度 wc->byte_len - 40；由库解析 L3 头构造回复 AH */
static struct ibv_ah *reply_ah(struct ibv_pd *pd, struct ibv_wc *wc, char *buf, uint8_t port)
{
    return ibv_create_ah_from_wc(pd, wc, (struct ibv_grh *)buf, port);
}
```

要点：UD QP 只需三步最小属性（对比 RC 的大量连接参数）；每个发送 WR 自带 AH、远端 QPN 与 Q_Key；单个消息不能超过 path MTU——规范要求超长的 UD 消息**既不发包也不报错**（C10-93.1.1）；接收缓冲前 40 字节保留给 L3 头，RoCEv2/IPv4 的 IP 头位于偏移 20（§A17.4.5.2），`byte_len` 总是包含这 40 字节；Q_Key 不匹配的报文在接收端被静默丢弃（C10-14）。`ibv_create_ah_from_wc()` 会解析该 40 字节区域（包括 IPv4/IPv6 判别，见 [04-work-requests-and-completions.md](04-work-requests-and-completions.md) 第 7 节）并构造回复用的 AH。

## 5. 用 librdmacm 简化建链（业界实践）

手工交换 QPN/PSN/GID 适合学习与调试；生产环境通常使用 librdmacm（rdma_cm），它以 IP 地址和端口号为接口，替你完成：按路由选择本端 RoCE 设备、端口与 SGID index（RoCE 版本由内核默认配置或 configfs 中的 `default_roce_mode` 决定）、解析 DMAC 与 hop limit、经 QP1 交换 IB CM 的 REQ/REP/RTU（在 RoCEv2 上就是普通的 RoCEv2 报文，§A17.6.1），并自动把 QP 迁移到 RTR/RTS〔业界实践: Linux 内核 rdma_cm 与 rdma-core librdmacm，以版本为准〕。同步接口的客户端骨架：

```c
#include <rdma/rdma_cma.h>
#include <rdma/rdma_verbs.h>
#include <stdio.h>
#include <string.h>

/* librdmacm 同步接口的 RC 客户端骨架：地址/路由解析、QP 状态迁移、CM 握手均由库完成 */
int cm_client(const char *server, const char *port)
{
    struct rdma_addrinfo hints, *res = NULL;
    struct ibv_qp_init_attr qia;
    struct rdma_conn_param cp;
    struct rdma_cm_id *id = NULL;
    struct ibv_mr *mr = NULL;
    struct ibv_wc wc;
    static char buf[64] = "hello via rdma_cm";
    uint8_t tos = 26 << 2;                     /* DSCP 26，写入 TOS 字节 */
    int ret = -1;

    memset(&hints, 0, sizeof(hints));
    hints.ai_port_space = RDMA_PS_TCP;         /* RC 语义 */
    if (rdma_getaddrinfo(server, port, &hints, &res))
        return -1;
    memset(&qia, 0, sizeof(qia));
    qia.cap.max_send_wr = qia.cap.max_recv_wr = 16;
    qia.cap.max_send_sge = qia.cap.max_recv_sge = 1;
    qia.sq_sig_all = 1;
    if (rdma_create_ep(&id, res, NULL, &qia))  /* 解析地址与路由（选 SGID、DMAC），建 PD/CQ/QP */
        goto out;
    rdma_set_option(id, RDMA_OPTION_ID, RDMA_OPTION_ID_TOS, &tos, sizeof(tos));
    mr = rdma_reg_msgs(id, buf, sizeof(buf));
    if (!mr)
        goto out;
    memset(&cp, 0, sizeof(cp));
    cp.responder_resources = 1;                /* 本端 Responder Resources */
    cp.initiator_depth = 1;                    /* 本端 Initiator Depth */
    cp.retry_count = 7;
    cp.rnr_retry_count = 7;
    if (rdma_connect(id, &cp))                 /* CM REQ/REP/RTU：QP1 上的 RoCEv2 MAD */
        goto out;
    if (rdma_post_send(id, NULL, buf, sizeof(buf), mr, 0) ||
        rdma_get_send_comp(id, &wc) <= 0 || wc.status != IBV_WC_SUCCESS)
        goto out;
    ret = 0;
    rdma_disconnect(id);
out:
    if (mr)
        rdma_dereg_mr(mr);
    if (id)
        rdma_destroy_ep(id);
    rdma_freeaddrinfo(res);
    return ret;
}
```

服务端对应地使用 `rdma_getaddrinfo()`（`hints.ai_flags = RAI_PASSIVE`）、`rdma_create_ep()`、`rdma_listen()`、`rdma_get_request()`、`rdma_accept()`。`rdma_conn_param` 中的 `responder_resources`/`initiator_depth` 对应 CM REQ/REP 的同名字段与 Modify QP 的 `max_dest_rd_atomic`/`max_rd_atomic`；`RDMA_OPTION_ID_TOS` 设置的是 TOS 字节，`RDMA_OPTION_ID_ACK_TIMEOUT` 可调整 Local ACK Timeout。CM 协议本身与 IP 寻址的细节见 [IP CM 与 rdma_cm](../04-connection-management/02-ip-cm-and-rdma-cm.md) 和 [IB CM 协议在 RoCEv2 上的应用](../04-connection-management/01-cm-protocol.md)。

## 6. 常见编程陷阱

| # | 现象 | 原因 | 依据 | 对策 |
|---|---|---|---|---|
| 1 | `ibv_modify_qp(INIT→RTR)` 返回 `EINVAL` | `ah_attr.is_global = 0`，没有填 GRH | §A17.4.2（GRH 标志恒为 TRUE）；Linux `rdma_check_ah_attr()` 对 RoCE AV 无 GRH 返回 `-EINVAL`〔业界实践〕 | 置 `is_global = 1` 并填写 `grh.dgid/sgid_index/hop_limit` |
| 2 | 建链“成功”，但所有请求超时（`IBV_WC_RETRY_EXC_ERR`） | GID index 选到 RoCE v1 表项、另一块网卡/VLAN 的地址、或两端一 v1 一 v2 | CA17-40、CA17-41：GID type 与报文协议不符时 DGID 检查失败，报文被丢弃 | 按类型与路由出口选择 RoCEv2 GID；两端协商同一类型 |
| 3 | RTR 失败，或大消息在对端出现长度错误 | `path_mtu` 超过端口能力，或两端不一致 | Verb 结果 “MTUCap of HCA port exceeded”；C9-92（报文长度须与 PMTU 相符） | `path_mtu ≤ active_mtu`，两端取较小值；`active_mtu` 由 netdev MTU 推导 |
| 4 | 偶发 `IBV_WC_RETRY_EXC_ERR`，或故障后很久才报错 | `timeout` 过小/过大；`timeout = 0` 关闭了传输定时器 | Ttr = 4.096 µs × 2^timeout，检测在 Ttr~4·Ttr（C9-140、C9-141） | 常用 14–18，并按 (retry_cnt + 1) × 4·Ttr 估算最坏失败时间 |
| 5 | 偶发数百毫秒延迟 | `min_rnr_timer = 0`（表示 655.36 ms），对端 RQ 偶尔为空 | Table 48 | 预投递足够的接收 WR；`min_rnr_timer` 取 12（0.64 ms）左右；`rnr_retry = 7`（无限）会掩盖问题 |
| 6 | RDMA READ/Atomic 以 `IBV_WC_REM_INV_REQ_ERR` 失败 | 本端 `max_rd_atomic` > 对端 `max_dest_rd_atomic`；或对端 QP 未使能 `IBV_ACCESS_REMOTE_READ/ATOMIC` | Table 61（Too many RDMA READ or ATOMIC；Unsupported OpCode）；§12.7.29–§12.7.30 | 交换并取较小值；设置对端 `qp_access_flags` |
| 7 | `IBV_WC_REM_ACCESS_ERR` | rkey 过期、远端地址/长度越界、MR 无对应远程权限 | C9-89、C10-57 | 检查 rkey/addr/len 与注册权限；不要在对端仍使用时注销 MR |
| 8 | RDMA READ 以 `IBV_WC_LOC_PROT_ERR` 失败 | 本地落地缓冲的 MR 没有 `IBV_ACCESS_LOCAL_WRITE` | §10.6.7.1、C10-55 | 为接收、READ、Atomic 结果缓冲注册 Local Write |
| 9 | 注册 MR 失败（`EINVAL`） | 设置了 `REMOTE_WRITE`/`REMOTE_ATOMIC` 却没有 `LOCAL_WRITE` | C10-48 | 同时设置 `IBV_ACCESS_LOCAL_WRITE` |
| 10 | UD 接收的数据“错位”或长度多出 40 | 没有为 L3 头预留 40 字节；把 IPv4 头当作从偏移 0 开始 | §11.4.1.2、§A17.4.5.2、§11.4.2.1 | 接收缓冲 = 40 + MTU；负载从偏移 40 开始；IPv4 头在偏移 20 |
| 11 | UD 发送“成功”但对端什么都没收到 | 消息超过 MTU（规范要求不发包也不报错）；或 Q_Key 不匹配（对端静默丢弃） | C10-93.1.1、C10-14 | 限制消息 ≤ path MTU；核对 Q_Key 与最高位规则（C10-15） |
| 12 | 大量 NAK-Sequence 或首包即失败 | `sq_psn` 与对端 `rq_psn` 不一致；未截断到 24 bit | §12.7.31；PSN 为 24 bit | 交换时写清字节序并 `& 0xFFFFFF` |
| 13 | Create AH / Modify QP 返回 Invalid Address Vector | SL 取了 8–15 | §A17.4.2、§A17.5.3–§A17.5.4 | 只用 SL 0–7 |
| 14 | 抓包看到的 DSCP 与预期不同 | 把 DSCP 值直接写进 `traffic_class` | Linux 把 `traffic_class` 当作完整 TOS 字节〔业界实践〕 | 设置 `DSCP << 2`；ECN 位一般由硬件按拥塞控制配置改写 |
| 15 | `ibv_post_send()` 返回 `ENOMEM` | 只投递 unsignaled WR，SQ 槽位无法回收 | §10.8.6、C10-108 | 每 N 个 WR 置一次 `IBV_SEND_SIGNALED` |
| 16 | 释放资源时报错（常见为 `EBUSY`） | 先释放了仍被引用的 PD/CQ，或 MR 上仍绑定 MW | C10-11.2.1、C10-18、§10.6.7.2.6 | 按 QP → MW → MR → CQ → PD → 设备的顺序释放 |
| 17 | `ibv_query_port()` 返回 `EINVAL` | 端口号用了 0 | libibverbs 端口号从 1 开始〔业界实践〕 | `port_num ≥ 1` |
| 18 | 期待 Port Change / Client Reregister 事件但永远等不到 | RoCEv2 端口不支持这两个事件 | CA17-31、CA17-32 | 用 `IBV_EVENT_GID_CHANGE`（Linux 扩展）或 netlink 跟踪地址变化 |

## 7. 要点回顾

- CA17-30：RoCEv2 HCA 必须采纳 §A17.5 的全部 Verbs 修改；修改围绕 AV 与“去除 IB 链路层/子网管理概念”。
- Query HCA：LID/LMC、VL、SM 信息与 4 个 CapabilityMask 位无效；新增端口类型值 “RoCEv2” 与 “RoCE Supported” 能力位。
- AV：GRH 标志恒真；SGID index 与 DGID 对所有目的地必填；GID type 决定协议；DLID 泛化为 Destination L2 Address（0xFFFFFFFFFFFF = 由实现解析）；SL 0–7 映射优先级，8–15 返回 Invalid Address Vector。
- 多播只用 MGID；Poll 的 UD 远端信息变为 SMAC + IPv4/IPv6/RoCE 标志；Get Special QP 只剩 QP1；Post Send 没有 Raw。
- RoCEv2 端口不支持 Client Reregistration 与 Port Change Event（CA17-31、CA17-32）。
- 编程关键：选对 RoCEv2 GID、`is_global = 1`、`path_mtu ≤ active_mtu` 且两端一致、PSN 互换、Initiator Depth ≤ 对端 Responder Resources、接收缓冲与 MR 权限正确。
- 生产环境优先使用 rdma_cm：它完成 GID 选择、DMAC 解析、CM 握手与 QP 状态迁移。

## 自测

<details><summary>Q1：在 RoCEv2 端口上调用 Get Special QP 申请 SMI QP（QP0）会怎样？对端发来 DestQP = 0 的报文呢？</summary>

答：Get Special QP 只适用于 QP1，申请 QP0 必须返回 “Invalid Special QP Type”（§A17.5.8）；DestQP = 0 的入站报文被静默丢弃（CA17-33）。
</details>

<details><summary>Q2：RoCE v1 与 RoCEv2 对“AV 中能否携带 L2 地址”的规定有何不同？Linux 用户态程序需要自己填 DMAC 吗？</summary>

答：RoCE v1 的 CA16-17 要求 AV 只含 GID、不得含 L2 引用；RoCEv2 把 DLID 泛化为 Destination L2 Address，允许携带已解析的 DMAC，也可以填 0xFFFFFFFFFFFF 让 Verbs 实现自行解析（§A17.4.2）。Linux 用户态 `ibv_ah_attr` 没有 DMAC 字段，由内核在 Create AH / Modify QP 时解析，程序无需也无法自己填。
</details>

<details><summary>Q3：示例程序为什么在交换连接参数之前、QP 还处于 INIT 时就投递接收 WR？</summary>

答：规范允许 INIT 下投递接收（C10-24）。提前投递保证 QP 一进入 RTR 就有接收 WQE 可用：若本端已处于 RTR/RTS 而 RQ 为空时对端发来 SEND，RC 响应方会回 RNR NAK，请求方要等 `min_rnr_timer` 指示的时间后重试，RNR 重试耗尽时还会报错。提前投递消除了这种竞态。
</details>

<details><summary>Q4：客户端的 RDMA WRITE 是 unsignaled 的，客户端如何知道它已完成？服务端为什么在收到 SEND 后就能读到 WRITE 的数据？</summary>

答：按 §10.8.6，取回同一 SQ 上后续 WR（这里是 SEND）的 WC 即确认之前的 unsignaled WR 已成功完成（C10-108）。RC 响应方按 PSN 顺序执行请求，WRITE 的数据先于其后的 SEND 被处理，所以 SEND 的接收完成出现时 WRITE 的数据已写入服务端内存。
</details>

<details><summary>Q5：一台机器的网卡同时有 RoCE v1 与 RoCEv2 两个同地址 GID 表项，对端只支持 RoCEv2。若本端误选了 v1 表项，会发生什么？</summary>

答：按 CA17-40 本端 QP 工作在 RoCE v1 模式，发出以太类型 0x8915 的帧；对端 RoCEv2 端口不处理这类帧（即便支持 v1，DGID 检查也会因 GID type 与报文协议不符而失败，CA17-41），报文被丢弃。本端只能靠超时重传，最终以 Transport Retry Counter Exceeded 失败。
</details>

## 参考

- 规范：IBTA Vol1 Rel 1.4 Annex A17 §A17.4–§A17.8（p.1936–1946）、Annex A16 §A16.8–§A16.9（p.1920–1926）、§10.8.6（p.554）、§11.2.1.2（p.588–594）、§11.2.2（p.598–602）、§11.2.5.2（p.611–618）、§11.2.6（p.622–624）、§11.3（p.661–663）、§11.4（p.663–683）、§11.6.3（p.690–695）、§12.7.29–§12.7.31（p.736–737）。
- 外部：rdma-core `libibverbs/verbs.h`、`librdmacm/rdma_cma.h`、`librdmacm/rdma_verbs.h` 及 man pages（`ibv_modify_qp(3)`、`ibv_query_gid_ex(3)`、`ibv_post_send(3)`、`ibv_poll_cq(3)`）〔业界实践: Linux rdma-core〕。
- 外部：Linux 内核 `drivers/infiniband/core/verbs.c`（`ib_resolve_eth_dmac()`、`rdma_check_ah_attr()`）〔业界实践: Linux kernel〕。
- 相关文件：[01-resource-model.md](01-resource-model.md)、[02-qp-state-machine.md](02-qp-state-machine.md)、[03-memory-management.md](03-memory-management.md)、[04-work-requests-and-completions.md](04-work-requests-and-completions.md)、[IP CM 与 rdma_cm](../04-connection-management/02-ip-cm-and-rdma-cm.md)、[Linux RDMA 软件栈](../07-engineering-practice/01-linux-rdma-stack.md)。
