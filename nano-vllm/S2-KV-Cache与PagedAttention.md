---
type: 学习成果
topic: nano-vllm S2
date: 2026-10-29
status: 已归档
source: 快照自 `vllm:note/S2-KV Cache与PagedAttention.md`（2026-10-29 收口归档）
tags:
  - vllm
  - nano-vllm
---

# S2 KV Cache 与 PagedAttention

kv_cache shape = (2, num_layers, num_blocks, block_size, num_kv_heads, head_dim)
`vllm:learn-nano-vllm/nanovllm/engine/model_runner.py`:354-361

kvcache = 2(k&v) * num_layers * num_blocks * block_size * 每个token的kv大小[num_kv_heads * head_dim * 2(bf16)] = 2 * 28 * 310 * 256 * 8 * 128 * 2 = 8.4765625GiB

num_kvcache_blocks=310的含义是：共有310个KV Cache块用于存储模型在推理过程中生成的键值对
total_gib = 11.629
free_gib  = 10.102
peak_gib  = 1.575
current_gib ≈ 1.14
gpu_memory_utilization = 0.9
num_kvcache_blocks = int(total * gpu_memory_utilization - used - peak + current) // block_bytes
11.629 × 0.9 - 1.527 - 1.575 + 1.14 ≈ 8.504 GiB
8.504 GiB / 0.02734375 GiB ≈ 311 blocks，实际用int向下取整为310
