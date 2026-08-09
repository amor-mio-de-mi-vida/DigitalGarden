---
title: SGLang PP × HiCache L3：状态发散与对齐范式
category: ai-infra
tags:
  - sglang
  - pipeline-parallelism
  - kv-cache
  - distributed-systems
draft: "false"
---

两份自包含的交互式幻灯片（方向键或 `空格` 翻页，键盘操作需先将焦点置于 iframe 内）。代码基线为 2026-08 的本地快照，所引行号均以此为准；链接指向上游 `main`。

**主报告 · PP × HiCache L3（25 页）** — [↗ 全屏打开](/static/slides/pp-hicache-l3-slides.htm)

<iframe src="/static/slides/pp-hicache-l3-slides.htm" style="width:100%;height:600px;border:1px solid var(--lightgray);border-radius:8px;" loading="lazy" title="PP × HiCache L3 社区问题与修复全景"></iframe>

**姊妹篇 · PP × 组合场景（9 页）** — TP / DP-attention / CP / EP × PP、PD 分离 × Mooncake、EAGLE 推测解码 — [↗ 全屏打开](/static/slides/pp-combination-slides.htm)

<iframe src="/static/slides/pp-combination-slides.htm" style="width:100%;height:600px;border:1px solid var(--lightgray);border-radius:8px;" loading="lazy" title="PP × 组合场景"></iframe>

## Background

PP 的每个 rank 各跑一个独立 `Scheduler`、各持一棵私有 radix tree 和一份 KV pool 分片，**彼此从不通信**。正确性完全依赖一条隐含契约：

> 输入相同 + 决策函数确定 ⇒ 所有 rank 的状态同构

HiCache 是三层 KV 缓存（L1 显存 / L2 主机内存 / L3 外部存储）。它将两路 IO 就绪状态引入了调度决策——L2 的 DMA ack 回收数、L3 的预取命中长度与下载进度——契约随即被打破：各 rank 的树开始发散（diverge），最终在 PP 边界上以形状不匹配或死锁的形式暴露。

社区在 2026-05 的周会上确定了策略：引入 L3 后，对 PP 追加补丁难以收敛，且 L3 不可用时 PP 缺少 fallback 路径，因此**先确保 L2+PP 的正确性，再处理 L3**。这条主线决定了后面所有 PR 的排序。

## Key points

### 三个贯穿全文的概念

| 概念 | 定义 | 能否进决策 |
| --- | --- | --- |
| **per-rank 量** | 取值由运行时物理时序决定的量：网络吞吐、磁盘延迟、DMA 完成时刻、GIL 争用、内存分配结果。同一逻辑步在不同 rank 上读到的值**必然不同**且不可复现 | ❌ 绝对不能 |
| **逻辑量** | 由确定性代码路径决定：第几个调度步、第 k 张 ack、请求序列里的第几个请求。全员必然一致 | ✅ 可以 |
| **协议量** | per-rank 量经过一次显式跨 rank 对齐（`all_reduce` / leader 广播）后得到的值 | ✅ 可以 |

`per-rank` 是代码注释里的原词：*"TP-MIN a **per-rank** HiCache DMA ack count … can **diverge across PP ranks**"*（`hiradix_cache.py:254`）。

**本文所述缺陷的共同根因是「per-rank 量直接进入决策」，修复方向统一为「先将其转换为协议量」。**

### 四类发散源

总纲 issue 归纳，全部是 per-rank 量：

1. **ack 回收数量** —— 每个调度步确认几张 DMA 完成，取决于本地 IO 进度
2. **L3 命中长度 / 下载进度** —— 各 rank 读取各自的 KV 分片，使用独立连接，且可能落在不同的存储节点
3. **host 内存分配成败** —— `write_backup` 在部分 rank 返 0
4. **LRU 时钟** —— `last_access_time` 用 `time.monotonic()`，各 rank 的时间戳存在微小差异 → 选出不同的逐出目标

### 三种崩溃形态

| 形态 | 签名 | 含义 |
| --- | --- | --- |
| A | `RuntimeError: shape '[N, -1, D]' is invalid` | 上下游 stage 对「本 batch 的 extend token 数」判断不一致。属 PP 维的**渐进式**发散，可能在运行数小时后才触发 |
| B | `IndexError: pop from empty list` | 沿用了 rank0 的 drain 计数，而本地队列长度更短 |
| C | `WorkNCCL Timeout(ms=600000)` | 某 rank 阻塞在永不满足的队列排水条件上，停止发起 p2p → 上游 SEND 阻塞 600 s 后被 watchdog 终止。TP 维通常**立即**失败，且极易被误判为网络故障 |

现场特征：空载亦会崩溃；`--hicache-size` 越大崩溃频率越高（80 GB 约每日一次，170 GB 单夜 4–5 次）；工单中记录过仅 PP0 持续泄漏约 12 GB/h、并被 OOM-kill 6 次的情形。

## Explanation in my own words

### 修复范式：四条对齐原语

> [!tip] P1 · 对称 MIN——最慢者获胜
> 「进度 / 容量 / 命中长度」类**数值**，跨 rank `all_reduce(MIN)`。MIN 是唯一所有 rank 均可满足的取值：共识长度以内的数据在每个 rank 都真实存在。代价是集体决策退化至最慢的 rank，即以缓存收益换取一致性。

> [!tip] P2 · Leader 决策 + pp_sync 链式广播
> 「是否执行 / 执行多少」类**控制**决策由 PP0 裁决后点对点链式下发。**不得使用 PP collective**：流水线各 stage 必须保持相位差，强制对齐将导致死锁。

> [!tip] P3 · 通信平面隔离
> 后台线程使用 gloo（CPU），与调度线程的 CUDA collective 分属不同通信平面；且**每条线程独立通信组**——同一个 communicator 被两个线程并发 `all_reduce` 会损坏或死锁。

> [!warning] P4 · 协议守恒——异常路径也要发消息
> 失败、终止、abort 路径均**不得少发**协议消息：等量 ack、每 rid 恰好一枚 sync token、共识后不二次 poll。MIN 投票靠「第 k 票配第 k 票」的次数配对，任何「提前 return」都会引入错位。

一句话概括：**IO 从未被对齐，被对齐的只是决策所用的数字。**

### L3 的修复难度高于 L2 的原因

L2 的 DMA ack 全部在调度线程一处回收，只需在 `check_hicache_events` 中加入协议即可。L3 的读数则产生于**三条后台线程**，在其传递至组批阶段之前，存在一段本地值未经对齐即可能被使用的窗口。因此主修复采用的结构是**由产生数据的线程自行投票**：探询线程经 PG1 对命中长度取共识，下载线程的 ack 经新增的投票线程走 PG2 对下载进度取共识，控制类决策沿用 pp_sync 由 PP0 裁决。

### chunked prefill × PP：正确引入 per-rank 量的案例

动态分块（`--enable-dynamic-chunking`，仅在 `pp_size > 1` 时生效）同样向 PP 引入了「耗时」这一 per-rank 量，但处理方式是正确的：PP0 启动时实测 128 个递减 chunk 拟合 $f(l)=al^2+bl+c$，`broadcast_object_list` 广播**原始样本**，全员跑同一个确定性 `lstsq`；运行时解 $f(L+x)-f(L)=T$ 反推每片 token 数，该计算为纯函数，运行时零通信。

其要点为**采样单点化、数据协议化、决策纯函数化**：per-rank 量仅在启动阶段进入系统一次。长 prompt 下解出的 chunk 尺寸按 $1/L$ 收缩，故以 `base/4` 作为硬下限约束，使流水线气泡与分片数量同时有界。

## Why it matters

### 生产可用性

- ✅ **PP + L2**（`--enable-hierarchical-cache`，不配 storage backend）：系列修复已合入，有 e2e KL 测试护航；动态分块同样可用
- ❌ **PP + L3**（`--hicache-storage-backend mooncake/…`）：官方口径 *"PP + L3 is not supported yet"*（2026-08-05），主修复及其伴生 PR 均未合并
- 若必须启用：跟进主修复分支并自行验证；`SGLANG_ENABLE_UNIFIED_RADIX_TREE=1` 缓解 LRU 时钟发散；调小 `--hicache-size`；监控三种崩溃签名

### 线上排障顺序

1. **先做分类**：存在 Python traceback → crash（形态 A/B）；无 traceback，且 GPU 利用率归零、约 600 s 后 watchdog 报错 → hang（形态 C）。两类问题根因不同
2. **crash：依异常类型定位**：`shape invalid` 查各 rank 的 `extend_num_tokens`；`pop from empty` 查各 rank 队列深度
3. **hang：先排除网络故障的误判**：本类缺陷的特征是「**单个** rank 阻塞在调度线程，其余 rank 阻塞于等待该 rank 的 p2p」。以 `py-spy dump` 逐 rank 查看调用栈即可区分
4. **通过开关二分定位**：未配置 storage backend 仍崩溃 → L2 维发散；调小 `--hicache-size` 后频率下降 → 与逐出压力相关

### 方法论的可迁移性

上述四条原语并非 HiCache 专属。任何满足「**多副本独立决策（SPMD）+ 资源异步就绪**」的系统都面临同一问题——MoE 的 expert 容量与 all-to-all、KV offload 与 PD 传输、投机解码的接受长度、分布式限流器等。判据可直接复用：**列出每个决策点的输入，标出其中由运行时物理时序决定的部分**，该清单即为系统的发散源清单；随后逐项确定其对齐方式——取 MIN、由 leader 裁决，或将决策收敛至单点。

真正的难点不在于修复单个决策点，而在于**证明该清单是完整的**；这也是主修复至今未能合并的原因。

## Related

- [[Infra/index|Infra]]
- [[CUDA/index|CUDA]]

## Reference

以下为两份 slides 内引用的全部外部链接。

### 总纲与问题报告

- [#22607](https://github.com/sgl-project/sglang/issues/22607) —— umbrella：PP + HiCache 兼容性总纲
- [#30158](https://github.com/sgl-project/sglang/issues/30158) —— GLM-5.2 FP8 / PP2+TP4+mooncake：空载亦崩溃；官方 *"not supported yet"* 表态出处
- [#28902](https://github.com/sgl-project/sglang/issues/28902) —— 只有 PP0 恒定泄漏 ~12 GB/h
- [#30476](https://github.com/sgl-project/sglang/issues/30476) —— PD+PP prefill 崩溃（abort 竞态）
- [#28429](https://github.com/sgl-project/sglang/issues/28429) —— 分配失败导致部分 rank 跳过 collective
- [#30760](https://github.com/sgl-project/sglang/issues/30760) —— 不需要 PP 的 TP-only 实证

### 已合并的修复

- [#27285](https://github.com/sgl-project/sglang/issues/27285) —— `pp_sync`：L2 事件消费数由 PP0 裁决后链式广播
- [#28916](https://github.com/sgl-project/sglang/issues/28916) —— 修复前一 PR 引入的 PP0 内存泄漏：为在飞发送设置硬上界
- [#29258](https://github.com/sgl-project/sglang/issues/29258) —— host 容量取 `ReduceOp.MIN`（各 stage 差 5.26%）
- [#31869](https://github.com/sgl-project/sglang/issues/31869) —— PD 队列与 abort：以共识结果为唯一依据，消除「共识后二次 poll」
- [#31443](https://github.com/sgl-project/sglang/issues/31443) —— sidecar 命中取前导连续成功段，避免空洞被计入安全前缀
- [#26923](https://github.com/sgl-project/sglang/issues/26923) —— `writing_check` 的 all_reduce 改为无条件执行
- [#29106](https://github.com/sgl-project/sglang/issues/29106) —— DeepSeek-V4 PP 下 SWA 分配与 layer 映射修正
- [#20460](https://github.com/sgl-project/sglang/issues/20460) · [#25887](https://github.com/sgl-project/sglang/issues/25887) · [#27366](https://github.com/sgl-project/sglang/issues/27366) —— 外围对齐修复

### 未合并的修复

- [#27010](https://github.com/sgl-project/sglang/issues/27010) ★ —— **L3 主修复**：`prefetch_sync_thread` + 等量 ack 协议 + pp_sync 扩展到预取控制队列
- [#31425](https://github.com/sgl-project/sglang/issues/31425) —— 内存分配两阶段事务：reserve → MIN 共识 → commit/abort
- [#22759](https://github.com/sgl-project/sglang/issues/22759) —— 逻辑 batch-step 时钟替换 `time.monotonic()`
- [#25148](https://github.com/sgl-project/sglang/issues/25148) —— `pp_prefix_len_cap`：各 stage 前缀能力取 min
- [#33473](https://github.com/sgl-project/sglang/issues/33473) —— write/load 完成数合并为一次 pp_sync，32K 输入 prefill 吞吐 +37%（494K→677K tok/s）

### chunked prefill × PP 动态分块系列

- [#15372](https://github.com/sgl-project/sglang/issues/15372) —— 画像采样 32→128 点；对齐下限 page → `max(page, 64)`
- [#16140](https://github.com/sgl-project/sglang/issues/16140) —— chunk 下限 `base/4`
- [#17198](https://github.com/sgl-project/sglang/issues/17198) —— 拟合丢弃首个样本（无 warmup 首跑偏慢）
- [#17339](https://github.com/sgl-project/sglang/issues/17339) —— DPA 下画像 forward 填 `global_num_tokens` 参与 MLP sync

### 组合场景（姊妹篇）

- [#16380](https://github.com/sgl-project/sglang/issues/16380) —— DSA prefill CP 与 PP 同开的传输层适配
- [#26227](https://github.com/sgl-project/sglang/issues/26227) —— decode 端开启 L3 prefetch

