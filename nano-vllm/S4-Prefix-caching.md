---
type: 学习成果
topic: nano-vllm S4
date: 2026-10-29
status: 已归档
source: 快照自 `vllm:note/S4-Prefix_caching.md`（2026-10-29 收口归档）
tags:
  - vllm
  - nano-vllm
---

# S4 Prefix Caching
> 2026-10-18 全本人结论
## 1. 源代码

| 位置 | 干什么 |
|---|---|
| `block_manager.allocate()` `:194-243` | 主逻辑。逐块算链式 hash → 查 `hash_to_block_id` → **三态**：① 未命中 → `KV_BLOCK_ALLOCATE` 从 free list 取新块；② 命中且块还在 `used_block_ids` → `KV_BLOCK_REUSE`，`ref_count += 1` 共享；③ 命中但块已回 free list → `KV_BLOCK_REACTIVATE`，直接复活 |
| `block_manager.may_append()` `:305+` | **谁把块注册进 cache**：decode 把某个块写满时 `block.update(h, token_ids)`，并写 `hash_to_block_id[h] = block_id`（`allocate` 里是 `:240-242` 同款）。不满的块不注册 |
| `block_manager.compute_hash(token_ids, prefix)` `:104` | 链式 hash：上一块的 hash 作为本块的 `prefix` 参与计算 → 前缀链不可拆 |
| `Sequence.num_cached_tokens` | **命中量的唯一真值来源**，在 `allocate()` `:214` 累加；`deallocate()` 会把它清 0，所以**跑完再读就读不到了**，必须挂钩子 |

---

## 2. 命中规则

**① 只有「整块」参与 cache，命中量是 256 的整数倍**

```python
# block_manager.py:196
h = self.compute_hash(token_ids, h) if len(token_ids) == self.block_size else -1
```

不满的最后一块 `h = -1`，而 `-1` 永远不会被注册进 `hash_to_block_id`（`:240` 有 `if h != -1` 守卫）→ **永远 miss**。
推论：prompt 总长 **< 256 token → 命中恒为 0**；命中量 = `256 × 连续匹配的满块数`，最多 `floor(prompt_len/256)×256`。

> 这也解释了 `experiments/05_block_manager.py` 为什么看不到效果：它的 prompt 只有 **25 / 24 token**，
> 连一个满块都没有，而脚本末尾的「观察要点」却写着「第二条请求应出现 REUSE」。
> `block_manager.py` 文件头注释（`:38-40`）其实已经写对了这条规则，是那个脚本没跟上。

**② 链式前缀：一旦某块没命中，后面即使内容相同也不复用**

```python
# block_manager.py:190
cache_miss = False  # 一旦某块未命中，后续块不再复用（前缀链断裂）
```

所以「共享一段后缀」完全没用，必须**从第 0 块开始逐块相同**。

**③ `can_allocate` 不看 cache（fork 是旧写法）**

```python
# block_manager.py:168
return len(self.free_block_ids) >= seq.num_blocks      # 没有扣掉可复用的块
```

命中前缀**不会降低准入要求**。显存紧张时会出现「本来能命中，却因为块不够进不来」，正好接回 S3 的结论
（KV 块容量是硬约束）。上游新版把这里改成返回 `num_cached_blocks` 并扣减 `num_new_blocks`，fork 还是旧写法。

**④ 惰性回收：块回 free list，但 `hash_to_block_id` 条目不清**

`deallocate()` `:253-267` 只做 `ref_count--` 并回收块，**不删 hash 映射**（文件头 `:33-36` 说明这是故意的）。
安全垫是 `:198` 的 `self.blocks[block_id].token_ids != token_ids` 比对——块被重新分配后 `reset()` 会把 `token_ids` 清空，
比对必然不等 → 走 miss。所以教案 09 里那个 issue「deallocate 后可能命中错误块」在这个 fork 里
**不会造成内容错误**；残留的只是 dict 里失效条目会累积（内存层面，不是正确性）。

---

## 3. 实测

```bash
cd /media/bed8oy/3T/01_workspace/vllm && source ./env.sh
python learn-nano-vllm/experiments/29_prefix_cache_ablation.py --jsonl logs/04_s4_prefix_cache.jsonl
```

> 原始数据落 `vllm/logs/`（与 `01_example_eager_teaching.log`、`02_s1_debug_tensor.log` 同目录），**不要写 `/tmp`**：
> 重启即丢、不进 Syncthing、笔记里引用不了。S3 的 jsonl 当初写在 `/tmp/s3.jsonl`，现在已经没了，
> 只剩笔记里的汇总表 —— `03_s3_param_ablation.jsonl` 是已知缺口。

三组对照，8 条请求，后缀各 64 token，每组跑 2 轮。
**第 1 轮 = 冷启动**（块还没进 cache，看命中与否）；**第 2 轮 = 重复提交同一批 prompt**
（上一轮的块已回 free list 但 hash 还在，专门看 REACTIVATE）。

| arm | 前缀 | prompt 长度 | 轮次 | `cached_tokens` 每序列 | 真实计算 prefill | 引擎口径 | 省下 | TTFT p50 | ALLOC/REUSE/REACTIVATE |
|---|---|---:|---|---:|---:|---:|---:|---:|---|
| **shared** | 1024（全共享） | 1088 | 冷启动 | 0, 1024×7 | **1536** | 8704 | 7168 | 84ms | 12 / 28 / 0 |
| | | | 重复提交 | 1024×8 | **512** | 8704 | 8192 | 32ms | 8 / 28 / 4 |
| **short** | 128（全共享） | 192 | 冷启动 | 0×8 | 1536 | 1536 | **0** | 68ms | 8 / 0 / 0 |
| | | | 重复提交 | 0×8 | 1536 | 1536 | **0** | 68ms | 8 / 0 / 0 |
| **random** | 1024（各不同） | 1088 | 冷启动 | 0×8 | 8704 | 8704 | **0** | 402ms | 40 / 0 / 0 |
| | | | 重复提交 | 1024×8 | 512 | 8704 | 8192 | 35ms | 8 / 0 / 32 |

> 本人实测 2026-10-18，原始数据 `logs/04_s4_prefix_cache.jsonl`（RTX 3060 / Qwen3-0.6B / `block_size=256` / 8 并发 / `output_len=8`）。
> 本次 `num_kvcache_blocks=300`，**不是 S2 算出的 310** —— 同一公式，只是当次 `peak − current` 不同（DEBUG + TEACHING 日志会抬高峰值）。
> 所以 310/300 都是"当次预算的结果"，**块数不是常量**，写报告时必须连同 `num_kvcache_blocks` 一起记。

结论：

1. **`shared` 冷启动**：第 1 条 `cached=0`（它就是那个"生产者"），后 7 条各命中 1024 → 省下 7168 token 的计算，
   prefill 时间 0.403s → 0.086s。**第一个请求永远要付全款。**
2. **`short` 两轮 `saved=0`**：192 token 的 prompt 即使**一字不差地重复提交两轮**，也一个 token 都省不下来——
   因为 192 < 256，凑不满一个整块。规则①最干净的证明（对比 `shared` 重复提交省下 8192）。
3. **`random` 冷启动 `saved=0`，重复提交时 8 条全部命中 1024**：块虽回了 free list，hash 条目还在 → 走 `REACTIVATE`（32 次）。
   这是「惰性回收」这条设计能直接换成速度的地方。

### REUSE vs REACTIVATE 差在哪（时序）

| | 触发场景 | 块当前在哪 | `ref_count` |
|---|---|---|---|
| `REUSE` | 上一个用这块的请求**还活着** | `used_block_ids` | +1（日志里能看到 1→2→…→8） |
| `REACTIVATE` | 上一个请求**已结束**，块回 free list 但 hash 还在 | `free_block_ids` | 重新置 1 |

想在**同一次 generate** 里看到 REACTIVATE，必须让前一条请求先结束（或被 preempt）。
`shared` 冷启动是 28 REUSE / 0 REACTIVATE，重复提交变成 4 REACTIVATE / 28 REUSE，就是这个区别。

---

## 4. 指标口径陷阱（重点）

`llm_engine.step()` 返回的 prefill token 数是：

```python
# llm_engine.py，注意 postprocess 已经先跑过了
num_tokens = sum(len(seq) for seq in seqs) if is_prefill else -len(seqs)
```

它统计的是 **prompt token 数 + 每条新生成的 1 个 token**，**完全没有扣掉 `num_cached_tokens`**。
三组冷启动那一轮对比一下就明白了：

| 组 | 引擎口径 | 真实计算 | prefill 时间 | TTFT p50 |
|---|---:|---:|---:|---:|
| `shared`（1024 全共享） | 8704 | **1536** | 84ms | 84ms |
| `short`（128 全共享） | 1536 | **1536** | 68ms | 68ms |
| `random`（1024 各不同） | 8704 | **8704** | 402ms | 402ms |

> 数字与 §3 同一次运行。每组只有 1 个 prefill step，所以 prefill 时间 ≈ TTFT；
> 跨轮跑会有 ±3ms 抖动，引用时以 jsonl 为准，不要混用不同轮的数。

这个口径**在两个方向上都会骗你**：

- `shared` vs `random`：引擎口径**都是 8704**，看齐了一模一样 → 你会以为 prefix cache 没生效；
  真实计算量差 **5.67×**（1536 vs 8704），TTFT 差 4.7×。
- `shared` vs `short`：引擎口径说 **8704 vs 1536**（"长前缀贵 5.7 倍"），但**真实计算量完全相同（都是 1536）**
  → 这个 5.7 倍是假的。谁要是拿 S3 harness 的 `prefill_tokens` 去论证"长前缀更贵"，结论就反了。

真正写着正确口径的地方在调度器里：

```python
# scheduler.py:86
num_batched_tokens += len(seq) - seq.num_cached_tokens    # ← "本步真正要算的 token 数"
```

**所以测 prefix cache 必须自己挂钩子读 `seq.num_cached_tokens`**（`29_...py` wrap 了 `BlockManager.allocate`），
用 S3 harness 的 `prefill_tokens` 会得出错误的对比结论。

---

## 5. 用于网关的结论（S4 的最终产出）

> Prefix cache 的命中粒度是 **256 token 的整块**，且要求**前缀链从第 0 个 token 起逐块一致**。
> 因此 cache-aware 路由只在「**共享前缀 ≥256 token 且逐块完全一致**」时才有收益；
> 共享几十个 token 的 system prompt、或只有片段相同的请求，收益为 **0**
> （`short` 组实测：192 token 的 prompt 重复提交两轮，`saved=0`）。
> 为这类请求牺牲负载均衡把流量打到同一实例，是净亏。

配套的两个量化事实：

- 收益上限 ≈ `(共享前缀长度/256) × 256 × (请求数 - 1)`：第一个请求之外才能省。
  实测 `shared` 冷启动省下 `4 × 256 × 7 = 7168`，正好是这个式子。
- cache 省的是**写 K/V + QKV 投影**（线性于 token 数），**省不掉对已缓存前缀的 attention 读取**。
  证据：`shared` 与 `short` 的真实计算量都是 1536，但 `shared` 的 TTFT 反而略高（84ms vs 68ms）——
  因为 `shared` 的序列要对 1024 个 cached K/V 做 attention，`short` 只对 192 个。
  所以**前缀越长，命中的相对收益（节省的时间 / 原本总时间）越小**，网关侧不该无条件偏好把长前缀请求并到一起。
