---
type: 学习成果
topic: nano-vllm S5
date: 2026-10-29
status: 已归档
source: 快照自 `vllm:note/S5-Prefill_Decode->TTFT_TPOT.md`（2026-10-29 收口归档）
tags:
  - vllm
  - nano-vllm
---

# S5 Prefill / Decode → TTFT / TPOT

TTFT = 排队 + 分词 + prefill forward + 采样首 token；会返回第一个token。
TPOT(time per output token) = (总时延 − TTFT) / (输出 token 数 − 1)
≈ (elapsed_s − prefill_time_s) / decode_steps
= (243.3ms − 84.2ms) / 7 = 22.7ms
ttft瓶颈是算力，因为prefill阶段要进行大量并行kv计算
tpot瓶颈在**大模型 + 足够大的 batch**情况下是带宽，因为每次生成 1 个 Token 都要将全量模型静态权重与历史 KV Cache 从全局显存（HBM）搬运至片上缓存（SRAM），但是并不需要很高的算力
但是TPOT 在本机 batch=8 时既不是算力（0.38ms）也不是带宽（6.1ms 下界），而是每步固定开销——实测 22.7ms，等效带宽只有 27%。

ttft和tpot会互相干扰：如果一张卡上正在全力decode时突然加入长prompt全量prefill，会造成tpot波动；反之，prefill等待decode会增加ttft

## S4 prefix cache 实测

TPOT 使用 `logs/04_s4_prefix_cache.jsonl` 中的
`(elapsed_s - prefill_time_s) / decode_steps` 计算；每个 round 有 7 个
decode steps。TTFT 受 prefix cache 显著影响，但 decode 阶段基本不受影响：

| arm | 轮次 | cache 命中 | TTFT p50 | prefill | elapsed | TPOT |
|---|---|---:|---:|---:|---:|---:|
| shared | 冷启动 | 0 + 1024×7 | 84.25 ms | 84.24 ms | 243.30 ms | 22.72 ms |
| shared | 重复提交 | 1024×8 | 32.34 ms | 32.34 ms | 190.73 ms | 22.63 ms |
| short | 冷启动 | 0×8 | 67.85 ms | 67.84 ms | 220.46 ms | 21.80 ms |
| short | 重复提交 | 0×8 | 67.48 ms | 67.48 ms | 219.50 ms | 21.72 ms |
| random | 冷启动 | 0×8 | 401.52 ms | 401.52 ms | 555.92 ms | 22.06 ms |
| random | 重复提交 | 1024×8 | 34.90 ms | 34.90 ms | 192.79 ms | 22.56 ms |

最能说明问题的是 `random` 冷启动与 `shared` 重复提交：前者没有命中
prefix cache，后者 8 条请求全部命中。TTFT 为
`401.52 / 32.34 = 12.42×`，而 TPOT 为
`22.06 / 22.63 = 0.97×`（反向表达为 `22.63 / 22.06 = 1.03×`）。
也就是说，prefix cache 几乎只加速 prefill/首 token，不会让 decode
阶段每 token 的生成速度提升一个数量级。

### 表里两个容易被忽略的数

**① shared 冷启动 84.25ms vs short 冷启动 67.85ms**：两组的**实际计算 token 都是 1536**（short 无命中；shared 是 8704 里命中 7168、剩 1536），但 shared 慢了 16.4ms。差在 **attention 仍要 attend 到那 7168 个已缓存的 KV**——缓存省的是「算」，不省「读」。

这也是「长前缀相对收益下降」的实测证据：0.6B 上 1024 前缀的读开销约 16ms，前缀越长这块越重，收益按 L/P 到 (L/P)² 缩水。

**② 吞吐 115 → 336 tok/s（2.9×）**：random 冷启动 115.1、shared 重复提交 335.6（8 并发 × 8 token ÷ elapsed）。TTFT 降了 12.4×，吞吐只涨 2.9×——因为分母里还有 7 步 decode 在跑。

> 一句话：prefix cache 改的是 TTFT（第一口气），不改 TPOT（每一口气）；吞吐的涨幅会被 decode 摊平。
