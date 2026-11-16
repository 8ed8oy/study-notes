# 自学笔记

我从 2026 年起的自学过程记录：课程主线笔记、源码阅读、实验记录。
不是教程，也不是教程的搬运——是**我自己读完、跑完之后写下的东西**，所以里面有错、有没做完的部分。

> **口径**：每篇都带 `status`。
> `已完成 / 已归档 / 已收口` = 我自己确认过的；`待做 / 待填 / 待开工` = 只有骨架，还没填；
> `待本人复核` = AI 起草、我还没逐条核对——**这类不要当结论用**。
> 文中的 `vllm:xxx`、`daily:xxx` 是本地仓库路径（`仓库名:相对路径`），对应的代码仓没有全部公开。

## 推理引擎与压测（主线）

| 篇目 | 内容 | 状态 |
|---|---|---|
| [nano-vllm 学习成果](nano-vllm/学习成果.md) | S1–S8 总览与产出 | 已完成 |
| [S1 请求生命周期](nano-vllm/S1-请求生命周期.md) | waiting → running → finished，step 里发生的三件事 | 已归档 |
| [S2 KV Cache 与 PagedAttention](nano-vllm/S2-KV-Cache与PagedAttention.md) | KV 显存算式 + `num_kvcache_blocks` 怎么算出来 | 已归档 |
| [S3 参数消融](nano-vllm/S3-参数消融.md) | 4 个旋钮对吞吐 / TTFT 的影响 | 已归档 |
| [S4 Prefix Caching](nano-vllm/S4-Prefix-caching.md) | 命中条件、三态事件、口径陷阱 | 已归档 |
| [S5 Prefill / Decode → TTFT / TPOT](nano-vllm/S5-Prefill-Decode-TTFT-TPOT.md) | 两个阶段与两个指标怎么对应 | 已归档 |
| [S6 引擎 / 网关边界（六问）](nano-vllm/S6-引擎与网关边界.md) | 哪些该在引擎做，哪些该在网关做 | 已归档 |
| [S7 闭卷自测（20 题）](nano-vllm/S7-闭卷自测.md) | 自己出的题，含答不出的那几道 | 已归档 |
| [S8 多卡并行（口述级）](nano-vllm/S8-多卡并行.md) | TP / PP 的取舍，只到口述 | 已归档 |
| [源码定位：用 grep 找一条链路](nano-vllm/源码定位-用grep.md) | 从日志关键字反推代码位置 | — |
| [T4 指标口径](vllm-benchmark/T4-指标口径.md) | TTFT / TPOT / 吞吐的口径与坑 | 已完成 |
| [L1 收口](vllm-benchmark/T8-L1收口.md) | vLLM Benchmark L1 的结论与失效样本 | 已收口 |
| [L1 / L2 任务单](vllm-benchmark/任务单-L1.md) | 实验开始前先定死的条件 | 已收口 / 待开工 |
| [T5 压测客户端](vllm-benchmark/T5-压测客户端.md) · [T6 主实验](vllm-benchmark/T6-主实验.md) · [T7 交叉核对](vllm-benchmark/T7-交叉核对.md) | 客户端、主实验、交叉核对 | 待本人复核 |

## 计算机网络

| 篇目 | 内容 | 状态 |
|---|---|---|
| [Chapter3 TCP](计网自顶而下/Chapter3.md) | 握手 / 挥手 / 拥塞控制 | — |
| [Chapter4 路由选择](计网自顶而下/Chapter4.md) | 核心路由算法 | — |
| [Chapter8 网络安全](计网自顶而下/Chapter8.md) | TLS 与密码学基础 | — |
| [实验卡：格式与清单](计网自顶而下/实验卡-00-格式与清单.md) | 「一屏实验卡」的模板与验收清单 | 进行中 |
| [实验卡：TCP 握手与 SYN 黑洞](计网自顶而下/实验卡-TCP握手与SYN黑洞.md) | 用 tcpdump 抓出「发 SYN 没回」 | 已完成 |
| [实验卡：DNS / keep-alive / TLS（骨架）](计网自顶而下/实验卡-DNS轮询与缓存TTL.md) | 三个实验只写了设计，没跑 | 待做 |

## 系统与工具

| 篇目 | 内容 |
|---|---|
| [linux 终端命令](linux/linux终端命令.md) · [分区与安装](linux/系统分区与安装.md) · [触控板](linux/触控板解决方案.md) · [软件清单](linux/推荐补充安装的专业软件.md) | 装机与日常排障攒下来的 |
| [K8s 4 小时任务卡](k8s/K8s-4h任务卡.md) | 还没开工 |
| [Go 并发笔记](go并发笔记.md) · [LeetCode](LeetCode.md) · [CS61A](CS61A.md) · [zju 命令速查](zju命令速查.md) · [Markdown 示例](Markdown.md) · [代码风格（cs61b）](代码风格与要求-cs61b.md) | 零散备忘 |

## 为什么把它们放一起

因为自学最容易骗自己：写过的笔记看起来像学会了，跑通一次的脚本看起来像懂了。
分状态标出来，是为了让「我到底做完了什么」一眼看得见——包括没做完的部分。