---
type: 任务卡
topic: K8s 最小可用（4 h）
date: 2026-11-12
deadline: 2026-11-29（W48；2026-11-15 本人从 11/21–22 改期）
status: 待开工
tags:
  - k8s
  - 自学
  - 求职
---

> ⚠️ **状态：待开工** —— 只有计划，还没开始。

# K8s 4 小时任务卡

> 目的不是「学会 K8s」，是**在简历上写一句真话**，并让面试里的 K8s 问题有东西可答。
> 依据：`0_规划/职业方向调研报告/字节与阿里招聘整理-2026-10-27` —— 8 条 JD 里 28 届唯一能投的是字节 **ByteHAS 实习**（无届次门，400–600/天，4 天/周），要求 **Go + K8s + 故障注入**。
> 深度上限：面经八股只到「**架构 / 副本扩容底层 / 金丝雀**」；调研报告 §3.1 的结论是「概念上知道**粘滞会话 / 自定义调度**即可，不必写 scheduler」。**不碰 Operator / Controller 源码。**

## 0. 部署对象：不要等网关

用 9 月的 **Go HTTP 服务 v0.1**（`go` 仓库，`go test -race` 通过那个）。
12 月网关做出来后重新部署一次，Dockerfile 与 YAML 直接复用，约 30 min。**不要为了「有东西可部署」去等网关。**

## 1. 四个小时怎么分

### H1 概念 + 起集群（1 h）

- Pod → ReplicaSet → Deployment 的从属关系；Service 与 Endpoint；ConfigMap / Secret
- `kind create cluster --name bench`
- **验收**：`kubectl get nodes` 有一个 Ready

### H2 把 Go v0.1 部署上去（1.5 h）

- 写 Dockerfile（多阶段构建，最终镜像尽量 < 30 MB）
- `kind load docker-image`（不是 push——本地集群拉不到远端 registry，这是最容易卡住的一步）
- Deployment + Service（`NodePort` 或 `kubectl port-forward`）
- **验收**：`curl localhost:<port>/...` 拿到 v0.1 的响应

### H3 探针与滚动更新（1 h）

- `readinessProbe` / `livenessProbe` 各配一个，写清「探针失败时分别会发生什么」
- `strategy.rollingUpdate` 的 `maxUnavailable` / `maxSurge`
- 改一版镜像 → `kubectl rollout status` → `kubectl rollout history` → **回滚一次**
- **故障实验**：把 readinessProbe 指向一个必定失败的路径，观察 Service 把 Pod 摘掉
- **验收**：能口述「滚动更新期间为什么可能丢请求、怎么消除」

### H4 资源与扩容（0.5 h）

- `requests`（调度单位）与 `limits`（硬顶）的区别；超 limits 会 OOMKilled
- HPA 按什么指标扩 —— **这一问的答案不在 K8s 文档里，在 benchmark 里**
- **验收**：能回答下面 Q3

## 2. 必须能答的六个问题（答不出就别写简历）

1. Pod 和 Deployment 什么关系？删掉一个 Pod 会发生什么？
2. 滚动更新怎么保证不中断？readinessProbe 和 livenessProbe 差在哪？
3. **HPA 你会用哪个指标扩容？**
4. `requests` 和 `limits` 差在哪？超出 limits 会怎样？
5. Service 怎么找到 Pod？Pod 换 IP 了怎么办？
6. ConfigMap 改了，正在跑的 Pod 会立刻生效吗？

### Q3 的答案（benchmark 给的）

> 不用 TTFT 当扩容触发指标。TTFT 是**排队的结果**，不是原因——等你看到它涨，请求已经排在队列里了。应该看**排队长度**或 **KV cache 占用率**。
> 证据：`T6-cold-c4` 的 `vllm:iteration_tokens_total` 显示 3 个请求被合并进同一步（816 = 3×272）；此时 TTFT 从 24.81 涨到 61.62，而系统吞吐只从 208 到 585。

这一条把 benchmark 和 K8s 焊在一起，是本科候选人很难说出来的判断。

## 3. 简历措辞（做完只准写这一句）

> 了解 K8s 基础，在 kind 上完成 Deployment/Service 部署、readinessProbe 配置与一次滚动更新与回滚。

**不许写**：「熟悉 K8s」「K8s 生产经验」。
理由：写了就要被追问副本扩容底层和生产集群规模，答不出会污染整份简历；写「在 kind 上做过」则问的是上面 1—6，全都有答案。

## 4. 明确不做

- Operator / CRD / Controller 源码 / Informer
- 调度器原理、CNI 插件、CSI 存储
- 真实集群、云厂商 K8s 服务
- CKA 之类的证书

## 5. 遗留：真正的短板不在这里

`0_规划/目标公司与团队清单` 已经写明短板是 **「K8s 生产级经验」**。
4 小时补不到，12 月也补不到，**只有 2027.1 的实习能补**。
所以被问「你们生产集群多大」时，正确回答是「我只在本地 kind 上做过，生产规模没接触过」——**装懂是唯一的死法**。