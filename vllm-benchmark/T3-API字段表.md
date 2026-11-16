---
type: 实验记录
topic: T3 API 字段对照
date: 2026-11-01
status: 待填
tags:
  - vllm
  - api
  - benchmark
---

> ⚠️ **状态：待填** —— 表结构在，数据还没填。

# T3 API 字段对照表

> 填法：每条 curl 打完，把结果填进对应行。**先看结构，再抄字段**。
> 拿状态码：`curl -si ... | head -1`，或 `curl -s -o /dev/null -w "%{http_code}\n" ...`。
> 原始响应想留档就存到 `vllm:logs/bench/probe/0N.txt`，不强制。

```shell
# 1 模型列表
curl -s http://127.0.0.1:8000/v1/models

# 2 非流式
curl -s http://127.0.0.1:8000/v1/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0.6b","prompt":"hi","max_tokens":4,"temperature":0}'

# 3 流式，默认（没有 usage）
curl -sN http://127.0.0.1:8000/v1/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0.6b","prompt":"hi","max_tokens":4,"temperature":0,"stream":true}'

# 4 流式 + include_usage（最后一帧 choices 是空数组）
curl -sN http://127.0.0.1:8000/v1/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0.6b","prompt":"hi","max_tokens":4,"temperature":0,"stream":true,"stream_options":{"include_usage":true}}'

# 5 chat 非流式，关 thinking
curl -s http://127.0.0.1:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0.6b","messages":[{"role":"user","content":"hi"}],"max_tokens":8,"temperature":0,"chat_template_kwargs":{"enable_thinking":false}}'

# 6 chat 流式（首帧只有 role，content 是空串）
curl -sN http://127.0.0.1:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0.6b","messages":[{"role":"user","content":"hi"}],"max_tokens":8,"temperature":0,"stream":true,"chat_template_kwargs":{"enable_thinking":false}}'

# 6b 同 6，加 include_usage（只想验证 usage 帧时用）
curl -sN http://127.0.0.1:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0.6b","messages":[{"role":"user","content":"hi"}],"max_tokens":4,"temperature":0,"stream":true,"stream_options":{"include_usage":true},"chat_template_kwargs":{"enable_thinking":false}}'

# 7 错误路径：错 model 名（看返回码和 body）
curl -si http://127.0.0.1:8000/v1/completions -H "Content-Type: application/json" \
  -d '{"model":"nope","prompt":"hi","max_tokens":4}' | head -5

# 8 /metrics 里有没有 TTFT 直方图（连 _count / _sum 一起抓）
curl -s http://127.0.0.1:8000/metrics | grep -E "time_to_first_token_seconds_(bucket|count|sum)" | head
```

## 1. 端点与响应

| #   | 请求                                                      | HTTP 码 | 响应形态                        | usage 在哪                                                        | 意外点                                                                                                                 |
| --- | ------------------------------------------------------- | ------ | --------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 1   | `GET /v1/models`                                        | 200    | 一整块 JSON                    | 无                                                               | 返回 `root`（权重路径）、`max_model_len`、权限等；`id` 就是启动时的 `--served-model-name`                                               |
| 2   | `POST /v1/completions` 非流式                              | 200    | 一整块 JSON                    | 响应末尾，`prompt/completion/total` 三个数                              | `prompt_tokens=1`（裸 prompt "hi"）；另有 `prompt_tokens_details`，未命中缓存时为 null                                            |
| 3   | `POST /v1/completions` 流式                               | 200    | SSE，4 帧 + `[DONE]`          | 无                                                               | 首帧就有 token（`"Question"`），不是空的；`finish_reason` 直接挂在最后一帧上，没有单独结束帧                                                     |
| 4   | `POST /v1/completions` 流式 + `include_usage`             | 200    | SSE，5 帧 + `[DONE]`          | 最后一帧（`[DONE]` 之前），`choices` 是空数组                                | 多出的那帧 `choices: []`，按 `choices[0]` 取会 IndexError                                                                    |
| 5   | `POST /v1/chat/completions` 非流式，`enable_thinking=false` | 200    | 一整块 JSON                    | 响应末尾                                                            | `prompt_tokens=13`，比 completions 的 1 大——chat 模板加了一段；`message` 里有 `reasoning` 字段，关掉 thinking 时为 null                 |
| 6   | `POST /v1/chat/completions` 流式                          | 200    | SSE，9 帧 + `[DONE]`          | 无；用 6b 加 `include_usage` 后，最后一帧 `choices` 为空、`prompt_tokens=13` | 首帧 `delta` 只有 `role`、`content` 为空串——**和 completions 相反**；9 帧 = 1 个空 role 帧 + 8 个 token 帧，最后一帧同时带文本和 `finish_reason` |
| 7   | `POST /v1/completions`，`model` 写错                       | 404    | 一整块 JSON（106 字节）            | 无                                                               | 是 404 不是 400；`type=NotFoundError`、`param=model`，不是参数校验错误                                                            |
| 8   | `GET /metrics`                                          | 200    | Prometheus 文本（`text/plain`） | 无                                                               | **不是 SSE**；TTFT 只有直方图分桶 + `_count`/`_sum`，拿不到精确分位数                                                                  |

## 2. 流式帧逐帧记录（选第 3 或第 4 条，抄前 4 帧 + 最后 2 帧）

| 帧序号   | `choices` 长度 | `text` / `delta.content` | `finish_reason` | 其他字段                                                          |
| ----- | ------------ | ------------------------ | --------------- | ------------------------------------------------------------- |
| 1     | 1            | Question                 | null            |                                                               |
| 2     | 1            | \n                       | null            |                                                               |
| 3     | 1            | What                     | null            |                                                               |
| …     |              |                          |                 |                                                               |
| 倒数第 2 | 1            | is                       | length          | usage: {prompt_tokens:1, completion_tokens:4, total_tokens:5} |
| 倒数第 1 | 0            |                          |                 |                                                               |

## 3. 错误码

| 场景       | 构造方式                            | HTTP 码 | 响应里的关键字段                                                              |
| -------- | ------------------------------- | ------ | --------------------------------------------------------------------- |
| 模型名不存在   | `"model":"nope"`                | 404    | `type=NotFoundError`、`param=model`                                    |
| JSON 非法  | 缺右括号                            | 400    | "message":"1 validation error:                                        |
| 参数越界     | `max_tokens` 超过 `max_model_len` | 400    | "message":"max_tokens=99999 cannot be greater than max_model_len=4096 |
| 限流 / 队列满 | 并发压满（T6 再补）                     |        |                                                                       |

## 4. 三个口径问题（自己的话答，不抄文档）

1. **流式的 TTFT 取哪一帧？** 是连接建立、首帧到达，还是首个非空 `text`/`content`？为什么？
2. **流式下输出 token 数从哪来？** 数帧、看 `usage`、还是 tokenizer 重算？三者会不会不一致？
3. **非流式和流式的 `usage` 差异是什么？** 为什么默认流式没有？

> 答：
> 1. 跳过 content 为空的帧，取第一个非空帧；不用首帧是因为chat 首帧是空 role 帧，会把 TTFT 少算一个 token 的时间
> 2.usage中最后一帧"total_tokens"给出了token数，chat首帧空，数会多数一个，但 tokenizer 重算可能会不一致
> 3.非流式会给出"prompt_tokens":1,"total_tokens":5,"completion_tokens":4,"prompt_tokens_details":null,"completion_tokens_details":null多个token统计，但流式只有"prompt_tokens":1,"total_tokens":5,"completion_tokens"三个。语义上，usage是终态信息；兼容上，默认关闭符合openai的规范

## 5. 结论（三行以内）
- 流式响应默认无usage，测tpot等需要手动`"stream_options":{"include_usage":true}`
- 测ttft需要跳过 content 为空的帧，因为chat首帧是role，无token
- 解析流式响应时先判 `choices` 是否为空再取 `[0]`，因为 `include_usage` 的最后一帧 `choices=[]`

## 6. 原始数据

- probe 输出：`vllm:logs/bench/probe/01_models.json` … `08_metrics.txt`（8 条各一份）
- server 日志：`vllm:logs/bench/server-20261101-r6.log`
