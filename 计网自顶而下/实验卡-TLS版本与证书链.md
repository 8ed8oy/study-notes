---
type: 实验卡
topic: TLS 1.2 与 1.3 握手对比、证书链
date: 2026-10-12
status: 待做
tags:
  - 计网
  - TLS
  - HTTPS
  - 一屏实验卡
---

> ⚠️ **状态：待做** —— 本页只有骨架，内容还没填。保留它是为了不假装做过。

# 一屏实验卡｜TLS 1.2 vs 1.3 与证书链（骨架）

> 格式见 [实验卡-00-格式与清单](计网自顶而下/实验卡-00-格式与清单.md)。**这是四张卡里唯一没有一手素材的一张**，需要新做。
> ⚠️ 下面所有输出都是**待做**，不要当成已完成的实验记录。

## 0. 一句话结论（待写）

（提示：要回答的是"1.3 相比 1.2 少了几次往返、证书为什么可信"。）

## 1. 实验元信息

| 项 | 值 |
|---|---|
| 环境 | 本机；注意系统 `openssl` 与运行时自带库可能**不是同一版本**（W40&41 踩过：系统 3.0.x，Node 内置 3.5.x） |
| 工具 | `openssl s_client`、`curl --tlsv1.2/1.3`、`tcpdump`、Wireshark（可选，看 `ClientHello` 扩展） |
| 对象 | 一个支持双版本的站点，如 `www.baidu.com:443`、`github.com:443` |

## 2. 问题（我要解释什么）

TLS 1.3 的握手在报文层面比 1.2 少了几步？证书链是怎么被验证到根的？`ClientHello` 里有哪些关键字段？

## 3. 预测（先写再跑）

> 如果 1.3 把密钥交换与认证合并、砍掉一次往返，那么：1.2 抓包应看到两步（ServerHello/Certificate 之后再 KeyExchange/Finished），1.3 的握手应在 ServerHello 之后基本一步完成；1.3 的 `ClientHello` 会带 `key_share` 提前发送公钥。

## 4. 操作（命令）

```bash
# 先确认仪器：系统 openssl 版本（避免"命令本身不工作"被当成网络故障）
openssl version -a

# 观察一：完整握手与证书链
openssl s_client -connect www.baidu.com:443 -servername www.baidu.com -showcerts </dev/null 2>&1 | head -60

# 观察二：强制版本，对比握手差异
openssl s_client -connect github.com:443 -servername github.com -tls1_2 </dev/null 2>&1 | grep -E 'Protocol|Cipher|Verify'
openssl s_client -connect github.com:443 -servername github.com -tls1_3 </dev/null 2>&1 | grep -E 'Protocol|Cipher|Verify'

# 观察三：抓包看握手次数与 ClientHello 字段
sudo tcpdump -ni any 'host github.com and tcp port 443' -c 40 -w tls.pcap
#   用 Wireshark 打开：看 ClientHello 的 supported_versions / key_share / SNI / ALPN
#   只有 tcpdump 时：
sudo tcpdump -ni any 'host github.com and tcp port 443' -A | grep -a -E 'SNI|h2|http/1.1' | head

# 观察四：证书有效期与链
echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

## 5. 观察（原始输出，待填）

```
（待跑）关键看点：
- Protocol: TLSv1.2 / TLSv1.3；Cipher 套件差异
- Verify return code（0 = 验证到根成功）
- 1.2 与 1.3 抓包里的报文轮次（ServerHello 之后各几条）
- ClientHello 中 supported_versions / key_share / SNI / ALPN
```

## 6. 解释（机制）

1. **握手往返**：TLS 1.2 完整握手 2-RTT；1.3 通过合并密钥交换与认证压到 1-RTT，会话恢复可 0-RTT（0-RTT 数据可重放，这是它的代价）。
2. **`key_share`**：1.3 的 `ClientHello` 里客户端**猜**组并提前发公钥，省一次往返；发错了服务端会让它重试（HelloRetryRequest）。**后量子组（ML-KEM）会让 `ClientHello` 明显变大**——这曾经把我误导到"MTU 黑洞"的假假设上，但那是**测量工具坏了**而不是机制不成立。
3. **证书链验证**：叶子证书 ← 中间 CA ← 根 CA（预置在信任库）；验证包含签名链、有效期、SAN 与域名匹配、吊销状态（OCSP/CRL）。
4. **SNI / ALPN**：SNI 让一个 IP 承载多站点（也是代理能按域名分流的前提）；ALPN 协商应用层协议（`h2` / `http/1.1`）。

## 7. 结论（待写）

## 8. 常见追问（预演）

- 为什么"证书错误"和"连不上"要分开看？——前者是握手已完成后的验证失败，后者连 `ServerHello` 都收不到。
- 0-RTT 的风险是什么？——可重放；只适合幂等请求。
- TLS 1.3 为什么不能抓明文看内容？——1.3 把证书也加密了（`EncryptedExtensions` 之后），只能看到 SNI/长度等元数据（除非用 keylog）。
- 如果 `openssl s_client` 报"参数不支持"，先做什么？——**先验证仪器版本**，别当成网络故障（W40&41 教训 1）。
