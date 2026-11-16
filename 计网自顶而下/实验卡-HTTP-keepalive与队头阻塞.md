---
type: 实验卡
topic: HTTP keep-alive 与队头阻塞
date: 2026-10-12
status: 待做
tags:
  - 计网
  - HTTP
  - 一屏实验卡
---

> ⚠️ **状态：待做** —— 本页只有骨架，内容还没填。保留它是为了不假装做过。

# 一屏实验卡｜HTTP keep-alive 与队头阻塞（骨架）

> 格式见 [实验卡-00-格式与清单](计网自顶而下/实验卡-00-格式与清单.md)。本文件是**骨架**：命令已备好，观察/解释/结论留到实际跑完再填。
> ⚠️ 下面所有输出都是**待做**，不要当成已完成的实验记录。

## 0. 一句话结论（待写）

（提示：要回答的是"HTTP/1.1 为什么在一条 TCP 连接上串行、浏览器为什么要开多条连接"。）

## 1. 实验元信息

| 项 | 值 |
|---|---|
| 环境 | 本机，用 Go 标准库起一个本地服务（顺带复用 Go 学习成果） |
| 工具 | `curl -v`、`tcpdump`、`ss` |
| 对象 | `http://127.0.0.1:8080/`（本地，避开代理干扰） |

## 2. 问题（我要解释什么）

同一条 TCP 连接上连续请求两次，连接会被复用还是重建？如果服务端故意慢，第二个请求会不会被第一个卡住？

## 3. 预测（先写再跑）

> 如果 HTTP/1.1 默认 keep-alive 且请求串行，那么：`curl -v` 对同一 host 两次请求应只看到 **一次** `Connected to`；第二次请求应复用连接、不发生新的三次握手。

## 4. 操作（命令）

```bash
# 起服务（Go 标准库，可加 -timeout 让某个 handler 慢 3s）
go run ./cmd/server            # 或用已有 Go HTTP 小服务

# 观察一：两条 URL 一次 curl（同一进程复用连接）
curl -sv http://127.0.0.1:8080/a http://127.0.0.1:8080/b 2>&1 | grep -E 'Connected|Re-using|GET /'

# 观察二：抓包看复用（在 loopback 上）
sudo tcpdump -ni lo 'tcp port 8080' -c 30

# 观察三：强制关闭连接对照
curl -sv -H 'Connection: close' http://127.0.0.1:8080/a 2>&1 | grep -E 'Connected|Closing'

# 观察四：连接数与状态
ss -tan | grep 8080
```

## 5. 观察（原始输出，待填）

```
（待跑）关键看点：
- 复用：只有第一次出现 Connected，之后出现 Re-using existing connection
- 对照：加了 Connection: close 后每个请求都 Connected
- 抓包：复用时刻不出现新的 Flags [S]
```

## 6. 解释（机制）

1. **keep-alive**：HTTP/1.1 默认持久连接，`Content-Length` 或 chunked 用来界定消息边界，连接归还连接池而不关闭；HTTP/1.0 需要显式 `Connection: keep-alive`。
2. **队头阻塞（应用层）**：一条连接上请求串行，前一个响应没完成，后面的请求发不出去——这就是浏览器同域开多条连接的原因（W38 抓包已观察到并行多连接）。
3. **与 HTTP/2 的对比**：多路复用把"多条连接"换成"一条连接上多路流"，但 TCP 层的队头阻塞仍在（TCP 是字节流，丢一个段会卡住所有流）——这是 HTTP/3 走 QUIC/UDP 的动机。

## 7. 结论（待写）

## 8. 常见追问（预演）

- keep-alive 的连接什么时候会被服务端关掉？——`Keep-Alive: timeout`、空闲回收、`max_requests`。
- 消息边界靠什么界定？——`Content-Length`；没有长度就必须 chunked 或关连接。
- 为什么抓包里第二次请求不出现 `Flags [S]`？——复用已有连接，直接在已建立的连接上发数据段 `Flags [P.]`。
