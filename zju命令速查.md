# zju 命令速查

> 本机（`inmmy`，Ubuntu 24.04）浙大校园网 / 代理 / RVPN 隧道的一站式 CLI。
> 源码 `~/zju-setup/`，安装到 `/usr/local/bin/zju`。
> **完整资料（环境事实、systemd 单元、权限、环境变量、故障复盘、待办）见
> `~/zju-setup/README.md` —— 那份是完整的，这份只留日常会用到的。**

---

## 0. 日常什么都不用做

全是 systemd 服务，开机自启、校内/校外自动切换、订阅自动更新。

| 情况 | 敲什么 |
|---|---|
| 看现在的状态 | `zju status` |
| 换机场节点 | `zju node` |
| 看流量 / 重置日 | `zju usage` |
| SSH 去寝室台式机 | `zju ssh` |
| 用台式机的 dsh 网页 | `zju dsh` |
| 隧道坏了 | `zju heal` |
| **酒店 / 机场 WiFi 登录页出不来** | `zju off`（关梯子再刷新/重开浏览器），登录完 `zju on` |
| **重新登录（约每两周）** | `zju login` |

> [!tip] **唯一要记时间的事**
> RVPN 登录态约两周过期。**过期前 1-2 天会弹桌面通知提醒你**，不用自己记。
> `zju status` 里也会显示「登录态年龄: N 天」。

---

## 1. 全部命令

```
zju / zju status        查看状态          zju watch [秒]    实时刷新面板
zju heal                隧道失灵急救      zju login         重新登录（验证码+短信）
zju usage [-r]          机场用量 / 重置日 / 到期日
zju campus|offcampus|auto   强制校内 / 强制校外 / 恢复自动
zju tun on|off          开 / 关 TUN（默认开着，别关）
zju off / on            关 / 开梯子（off 停 mihomo：TUN / fake-ip DNS / :7890 全没）
zju test [主机]         校内连通性（直连 vs 隧道）对比
zju logs [n]            zju-connect 日志

zju ssh [主机] [用户] [ssh参数...] [-- 远端命令...]
zju dsh [本地端口] [--no-open]      zju dsh url

zju node                看分组和节点    zju node <名称|序号>   切节点
zju node auto           切回自动选择    zju node test          全部测速
zju groups              列策略组        zju dash               面板地址

zju update              重新拉订阅 + 生成配置
zju sub [show|add|url|rm|edit]       订阅管理
```

`zju` = `zju status`；`zju help` 打印更全的表。

---

## 2. 状态与急救

```bash
zju status
```

**只盯两行**：

| 行 | 看什么 |
|---|---|
| **隧道可用性** | 只有这行代表"真的能用" |
| 校园网组 | `DIRECT` = 校内直连／`ZJUconnect` = 走隧道（探针自动管） |

`zju status` 还会显示当前 AP 和判据，例如：

```
✔ 网络位置: 校内
• AP: ZJUWLAN-Secure(7C:1E:06:16:3B:63)
• 判据: 10.10.98.98:443   [探针]
```

> [!danger] 「运行中」≠「能用」
> 隧道进程活着、端口在监听，**会话也可能已经过期**，此时访问校内全失败。
> 坏了先 `zju heal`；heal 也救不回来就是会话过期 → `zju login`。

```bash
zju heal        # 重启隧道，并轮询最多 25 秒确认真的恢复
zju watch 2     # 挂一个终端窗口，每 2 秒刷新
```

---

## 3. 校内 / 校外切换：自动的

探针**每 60 秒**判定一次；换 WiFi / 断连时 NetworkManager 会立刻再触发一次。
判定靠**真去连校内 IP**（不看 DNS），所以隧道挂了也能判对。

```bash
zju campus        # 强制当作校内（很少用）
zju offcampus     # 强制当作校外（校园网抽风时应急）
zju auto          # 交还自动控制 ← 手动强制过就记得跑这个
```

---

## 4. 机场节点

```bash
zju node          # 看当前选择 + 节点列表（带序号和延迟）
zju node 14       # 按序号切
zju node 新加坡06  # 按名字模糊匹配
zju node auto     # 切回「自动选择」（按延迟自动选优）
zju node test     # 全部节点测速（慢，几十秒）
```

> [!tip] 日常就保持 `机场 → 自动选择`
> 只有某条线路明显抽风时才手动指定具体节点。选了会记住。
> 图形界面：<http://127.0.0.1:9090/ui/>

分组关系（**「校园网」组别手动改**，探针在管）：

```
机场 (Selector)          ← zju node 操作的是它，日常不用动
  ├─ 自动选择 (URLTest)   ← 默认选它，按延迟自动挑
  ├─ 宝贝云 (Selector)    ← 订阅自带的组
  └─ 44 个具体节点

校园网 (Selector)        ← 探针管，别动
  ├─ DIRECT
  └─ ZJUconnect
```

---

## 5. 用量 / 重置日 / 到期日

```bash
zju usage            # 详细：进度条 + 上传/下载/剩余/重置/到期 + 机场公告
zju usage --compact  # 一行一个订阅
zju usage -r         # 先重新拉订阅再显示
```

`zju status` 里也会自动附一行 compact 版。

---

## 6. 访问校内网站

**什么都不用配** —— TUN 在路由层分流，浏览器 / 命令行 / docker / 虚拟机全都直接可用。

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://www.zju.edu.cn   # 200
zju test cc98.org      # 对比「直连」和「强制走隧道」的状态码 / 耗时
```

---

## 7. SSH 到校内机器

```bash
zju ssh                 # 默认寝室台式机（用户取 ~/.ssh/config）
zju ssh desktop         # 用 ssh config 里的别名
zju ssh 10.10.98.98     # 校内其他机器
```

自动选路（校内直连 / 校外走 SOCKS5）。**优先用它而不是裸 `ssh`** ——
TUN 下裸 ssh 会出现假可达（台式机没开机也"连得上"）。

**跑远端命令要用 `--` 隔开**（ssh 语法是「选项 → 主机 → 命令」）：

```bash
zju ssh desktop -- hostname
zju ssh desktop -- 'uptime -p'
zju ssh desktop 'ls -l /tmp'      # 带空格/元字符的会自动当命令，不用 -- 也行
```

额外参数原样透传给 ssh，端口转发直接写：

```bash
zju ssh desktop -L 3081:127.0.0.1:3080         # 前台转发，Ctrl+C 断开
zju ssh desktop -N -f -L 3081:127.0.0.1:3080   # 后台常驻
```

> [!warning] `zju ssh desktop hostname` 会被当成「用户名 = hostname」
> 第二个位置参数按约定是**用户**。要么用 `--`，要么写成
> `zju ssh desktop 'hostname -f'` 这种一眼是命令的样子。

---

## 8. 用台式机的 dsh 网页

台式机的 dsh 只监听它自己的 `127.0.0.1:3080`，每次启动会把**带 token 的网址**打进
journal。`zju dsh` 把「建隧道 → 抓网址 → 换端口 → 开浏览器」全做了：

```bash
zju dsh              # 默认本地 3081，自动开浏览器
zju dsh 13080        # 指定本地端口
zju dsh url          # 隧道已在，只重新抓网址并打开
zju dsh --no-open    # 只打印不打开
```

> [!danger] 本地端口别用 3080
> **本机自己的 `dsh web` 就占着 `127.0.0.1:3080`**，所以默认用 3081。

关闭已有转发：

```bash
ss -tlnp | grep 3081
pkill -f 'ssh.*-L 127.0.0.1:3081'    # 只在交互 shell 里这么写
```

---

## 9. 登录（约每两周一次，唯一的人工环节）

```bash
zju login
```

必须在**你自己的终端**里跑：浏览器点图形验证码 → 终端输短信码 →
看到 SOCKS5 监听就 `Ctrl+C`（脚本会自动把服务交回 systemd 托管）。

> [!important] 为什么不能自动化
> 图形验证码要鼠标点、短信码要打进终端 —— 学校刻意设计的反自动化。
> 登录态在 `/etc/zju-connect/client_data.json`，**约两周有效**。
>
> **现在会主动提醒**：
> - 登录态超过 **12 天** → 弹桌面通知（每天最多一条）
> - 校外隧道起不来、自愈也无效 → 弹紧急通知（6 小时最多一条）
>
> `zju login` 会先**备份**旧登录态（`client_data.json.bak-<时间戳>`），
> 中途失败可以回滚。

---

## 10. 服务与 TUN

```bash
zju tun on       # 开 TUN（默认就是这个状态）
zju tun off      # 关 TUN：只剩 127.0.0.1:7890，得手动给程序配代理
```

> [!note] 平时开着
> TUN = 全局透明分流，不用给任何程序配代理。
> **关掉后外网只有配了代理的程序能用**，而且校外访问校内会直接失效。
> 只有两种时候该关：① 外面要先登录的 WiFi（`zju off`）
> ② 排查网络问题。

### 关梯子：`zju off`

```bash
zju off    # 停 mihomo：TUN / fake-ip DNS / :7890 一起消失 + 清 DNS 缓存
zju on     # 开梯子
```

`zju off` **不碰 RVPN 隧道**（`zju-connect` 没有需要手动停的场景，闲着不碍事），
只关梯子 —— 就这一个语义，不用记别的命令。

> [!note] 外面联网是手动的
> 不做"自动检测门户 / 自动关梯子"。外面的网络太杂（门户 / 插广告 / 要客户端认证），
> 自动动作容易帮倒忙，自己判断、自己敲 `zju off` / `zju on` 更可控。

### 校园网连哪个 SSID

| SSID | 认证方式 | 要网页门户吗 |
|---|---|---|
| **`ZJUWLAN-Secure`** | IEEE 802.1x | **不要** ← 推荐，零交互 |
| `ZJUWLAN` | 开放 + 网页门户 | 要（`zju off` 后浏览器登） |
| `ZJUWLAN-NEW` | 网页门户 | 要 |
| 手机热点 | — | 不要 |

已设优先级：`Secure` = 10 > `ZJUWLAN` = 0 = 手机热点（正常不会抢 Secure）。

---

## 11. 订阅

```bash
zju update                    # 重新拉订阅 + 生成配置（每 12h 也会自动跑）
zju sub                       # 看当前订阅（token 自动打码）
zju sub url 宝贝云 '<新链接>'  # 只换链接
zju sub edit                  # 直接编辑订阅列表
```

> [!warning] 贴输出前留意 token
> `zju sub` 默认打码；`zju sub show` 是明文，别往聊天 / 截图里贴。
> token 泄露就去机场后台重置链接，然后 `zju sub url ... && zju update`。

---

## 12. 出问题了

| 症状 | 先敲什么 |
|---|---|
| 校外连不上校内 / 台式机 | `zju heal`；25 秒没好 → `zju login` |
| 校内网站打不开 | `zju status` 看校园网组；判错了 `zju campus` |
| 国外网站打不开 | `zju node` 看延迟 → `zju node auto`；还不行 `zju update` |
| 全部网站都不通 | 先 `zju status` 看 mihomo 在不在；要干净网络排查用 `zju off` |
| 酒店 / 机场 WiFi 登录页出不来 | `zju off` → 刷新/重开浏览器登录 → `zju on` |
| 提示要登录 | `zju off` 后过门户；RVPN 会话过期则 `zju login` |
| 不知道怎么回事 | `zju status` → `zju logs 100` → 翻 `~/zju-setup/README.md` |

> [!tip] 关掉梯子后页面还是打不开？
> 浏览器缓存了 fake-ip（`198.18.x.x`）。**重启浏览器**，或 `Ctrl+Shift+R` 强刷。
> `zju off` 已经顺手清过系统 DNS 缓存，剩下的就是浏览器自己那份。

> [!tip] 国外网站"时好时坏"多半是节点拥塞，先分层测（不用改配置）
> ```bash
> curl -x socks5h://127.0.0.1:7890 -o /dev/null -w '%{http_code}\n' https://www.google.com
> ```
> 通 → 节点问题，`zju node auto`；不通 → 才考虑配置。

---

## 13. 红线

- **别启动 Clash Verge**：会抢 `7897` 并把 GNOME 系统代理指过去，
  **绕过 mihomo 导致校内分流失效**。不需要它了：
  `sudo apt remove clash-verge`
- **别手动改「校园网」策略组**，探针在管；要改就用 `zju campus` / `zju offcampus`
- **排查"没网"用 `zju off`**，别只 `zju tun off`（后者 mihomo 的 DNS 和 `7890` 都还在）
- **GNOME 系统代理保持 `none`**（TUN 模式下不需要）
- **别把自愈参数调激进**：每次重启都发一次认证请求，学校会封号 / 封 IP
- 改了 `zju` CLI 之后要重装：
  `sudo install -m 755 ~/zju-setup/zju /usr/local/bin/zju`
- 往聊天 / 截图贴 `zju sub` 输出前留意 token

---

> 想深挖（端口占用、systemd 单元、权限、环境变量、每一步踩过的坑）→
> **`~/zju-setup/README.md`**
