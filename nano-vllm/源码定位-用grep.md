## 第一步：先看清你答的是什么类型的问题

题目问：「这是**谁**做的决定——模型、调度器，还是块管理器？」

这是个**概念问题**，听起来可以凭理解编一个答案。但源码里唯一有确定答案的版本是个**机械问题**：

> `num_cached_tokens` 这个字段，**在哪个文件的哪一行被赋值**？

机械问题只有对错，编不出来。**你的错不是不懂 prefix caching，是把机械问题当成概念问题答了。**

翻译公式：

```
题目里的名词  →  它在代码里的字段名  →  grep 这个名字的「写操作」
```

## 第二步：四步定位（含真实输出）

**① grep 题目里的原词**

```bash
$ grep -rn "cached_prefix_tokens" nanovllm/ --include=*.py
nanovllm/engine/scheduler.py:96:  "seq_ids=%s cached_prefix_tokens=%s",
```

只有 1 处，而且在 f-string 里。**这是陷阱**：它是"打印的人"，不是"决定的人"。

> 规则：grep 到的第一处如果只是 log/print，**别停**，去找真正持有这个值的变量。日志名和字段名经常不一样。

**② 换成字段名再 grep**

```bash
$ grep -rn "num_cached_tokens" nanovllm/ --include=*.py
nanovllm/engine/block_manager.py:25:  allocate(seq) → 分配 blocks, 填 seq.block_table, 设 seq.num_cached_tokens
                                                              ↑↑↑ 第一行就是答案
```

25 处命中。**但你连读都不用读——第 1 条命中就是答案。**

这个教学版仓库有别的仓库没有的东西：**文件头自带接口索引**。`block_manager.py` 第 20–31 行直接画出了：

```
can_allocate(seq)  → 检查 free blocks 够不够
allocate(seq)      → 分配 blocks, 填 seq.block_table, 设 seq.num_cached_tokens
一条请求的生命周期: can_allocate → allocate → [may_append × N 次 decode] → deallocate
```

> 规则：**先读文件头 40 行**。别人替你写好的目录，比你自己摸索快十倍。这里前 40 行就是这个引擎的架构图。

**③ 只留「写」操作，把噪音滤掉**

```bash
$ grep -rn "num_cached_tokens *+\?=" nanovllm/ --include=*.py
block_manager.py:214:  seq.num_cached_tokens += self.block_size   ← 生产
block_manager.py:280:  seq.num_cached_tokens = 0                  ← 回收时重置
sequence.py:44:        self.num_cached_tokens = 0                  ← 初始化
```

三个写点，性质完全不同：

| 语句形态 | 含义 |
|---|---|
| `self.x = 0` | 初始化 |
| `seq.x = 0` | 重置/回收 |
| `seq.x += ...` | **增量累加 = 真正在"决定"** |

> 规则：**`+=` 是生产，`= 0` 是初始化。** 你要找的是 `+=`。

到这里就是 `block_manager.py:214`。**整个过程约 30 秒，没有读过一行逻辑。**

**④ 用 trace 事件名交叉验证**

```bash
$ grep -rn "KV_BLOCK_ALLOCATE" nanovllm/ --include=*.py
nanovllm/engine/block_manager.py:206:  "KV_BLOCK_ALLOCATE",
```

你日志里那条 `TEACHING/KV_BLOCK_ALLOCATE` 事件，源码位置就在 L206——**离你找到的 L214 只有 8 行**。两个独立方法指向同一区域，可以确信没找错。

> 规则：**教学日志里的事件名就是源码里的搜索键。** 这是这个 fork 最值钱的地方：先用日志看到"发生了什么事"，再用事件名 grep 回源码看"怎么发生的"。

## 第三步：为什么你的直觉指向"调度器"（这一步很重要）

你的直觉**不算乱猜**。向外扩一层看：

```bash
$ grep -rn "can_allocate\|\.allocate(" nanovllm/ --include=*.py
block_manager.py:166:  def can_allocate(self, seq)
scheduler.py:65:       cannot_alloc = not self.block_manager.can_allocate(seq)
scheduler.py:85:       self.block_manager.allocate(seq)
```

**调度器确实是调用者**（`scheduler.py:85`）。所以你的答案差的是**一层**，不是方向：

| 问法 | 答案 |
|---|---|
| 谁**触发**分配？ | Scheduler（它决定何时、是否值得分配） |
| 谁**计算**命中多长？ | BlockManager（`compute_hash` + `hash_to_block_id`） |

题目问"谁做的**决定**"，问的是计算方。

> 规则：**"谁"永远有两种读法——谁触发（caller）vs 谁计算（callee）。** 面试被追问时，答完 caller 一定要补 calcalee，否则听起来对但不严密。这条在 vLLM 真实代码里会一直用到。

## 第四步：这套方法的完整形状

```
1. 题目名词 → grep 原词           （常见结果：只找到打印点，别停）
2. 换成字段名 → grep              （先看文件头注释，通常是索引）
3. 过滤「写」操作：+= / = 0       （+= 才是生产，=0 是初始化）
4. trace 事件名 → grep 源码        （交叉验证，且能定位到事件发生的精确位置）
5. 向外扩一层：谁调用它            （回答"谁负责"必须走这一步）
```

**这套流程对 S1 是可复用的。** 比如"Scheduler 怎么决定 prefill 还是 decode"，同样四步：grep `is_prefill` → 收敛到 `schedule()` → 看 `waiting/running` 两个队列的分支。

## 立刻可练的两道（用上面的流程，控制在 5 条 grep 内）

1. **谁决定一个 block 被回收？**（提示：`deallocate` 的调用方）
2. **谁决定 `engine_step=1` 走 prefill 而不是 decode？**（提示：`schedule()` 里的分支条件）

每题你只需交：`文件:行号` + 一句话。**不许读整个文件。**