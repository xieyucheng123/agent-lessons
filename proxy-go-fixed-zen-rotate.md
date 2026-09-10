# Go 固定出口 + Zen 轮换出口分流（sing-box 双入口 + 本地 L7 网关）

> 背景：同一台机器同时调官方 Go（要固定出口 IP，便于归因/白名单）和官方 Zen
> （要轮换出口 IP，扛限流/容灾），但两者**同域名 `opencode.ai`、仅 HTTP 路径不同**
> （`/zen/go` vs `/zen`），L4 分流写不出两条规则。
> 本篇记录 .41 机器的实战解法：`127.0.0.1:18080/18081` 本地网关按端口分流。
> 前置篇：[tokyo-proxy-best-practice.md](tokyo-proxy-best-practice.md)（单出口 baseline）。
> 所有敏感值均为占位符（`<...>`），真值只在私仓 `agent-secrets/proxy/go-zen.env`，不进 git。

## 0. 架构

```
opencode provider opencode-go ──► http://127.0.0.1:18080/zen/go/v1 ─┐
opencode provider tokyo ────────► http://127.0.0.1:18081/zen/v1 ────┤
                                                                   ▼ L7 网关 (stdlib, systemd 常驻)
                        18080 Go-fixed ──► 127.0.0.1:2081 (sing-box mixed 入口#2) ──► outbound [proxy] 固定出口
                        18081 Zen-rotate ─► 127.0.0.1:2080 (sing-box mixed 入口#1) ──► outbound [pool-auto] urltest 轮换
                                                                                              │
                                                           ┌────────────────────────┬───────────┴────────────┐
                                                           ▼                        ▼                        ▼
                                              <VM_PROXY_IP>:8443            :9443                    :10443
                                              (eg-in-1 → eg-out-1)          (eg-in-2 → eg-out-2)     (eg-in-3 → eg-out-3)
                                              私网 IP #1                    #2                       #3
```

- Go：固定单一出口（本环境实测是 vm-proxy 全机唯一的 v6 地址，见第 5 节）。
- Zen：v4 三池轮换，`urltest` 每小时测速 + `tolerance` 防抖（坏 IP 最多赖 1h，
  短连接下次重选，另有整池轮换兜底）。

## 1. 为什么必须 L7 网关（两条死路先排除）

1. **sing-box 只能 L4 分流**：`CONNECT` 里只有 `host:port`，TLS 加密后看不到 HTTP 路径，
   同域名的 `/zen/go` 与 `/zen` 写不出两条 route 规则。
2. **opencode Provider 无单路代理选项**：schema 只有 `baseURL/timeout` 等，
   全局 `HTTP(S)_PROXY` 一改全家一起走（官方 Network 文档只给了全局项）。
3. 所以解法：两个 provider 指向**本地两个不同端口**，网关按端口决定进哪个 sing-box 入口。
   备选是给 Go/Zen 配独立域名（要动服务端/Nginx），当时原则是 OCI 零改动，选了本地网关。

## 2. sing-box：加一个固定入口 + 首位 inbound 路由

`~/docker/sing-box/config.json` 在原 2080 基础上加：

- `inbounds` 加一个 `type: mixed` 入口，端口 `2081`，
  **`listen` 必须写 `0.0.0.0`**（容器内写 127.0.0.1 会导致 docker-proxy 转发失败，见 §6.2），
  对外收口靠 compose 映射 `127.0.0.1:2081:2081`。
- `route.rules` **首位**加 `inbound: [<2081 入口 tag>] → outbound: proxy`（固定出站），
  原有 `opencode.ai → pool-auto` 等规则顺序不动。
- `pool-auto`（urltest）：`interval: 1h`（测速间隔），`tolerance` 防抖，
  **`idle_timeout` 必须 ≥ `interval`**（曾配成 interval 1h + idle_timeout 30m，
  sing-box 直接 FATAL 死循环重启，见 §6.1）。

验证：经 2081 的请求走 `outbound[proxy]` 固定出口，经 2080 的走 `[pool-auto]` 轮换，
看 sing-box 增量日志的 outbound 名即可区分。

## 3. L7 网关 `gateway.py`（18080 / 18081）

- 纯 stdlib（`http.server` 多线程），无第三方依赖，`py_compile` 过后再上 systemd。
- `CONNECT` 建隧道双向转发；普通请求透传，**流式/SSE/chunked 不缓冲**（推理是长流，不能攒包）。
- 自带 `/__egress` 探针口，回 `{"port": ..., "via": ...}`，证明请求走了哪个上游。
- systemd 单元常驻 + 开机自启；网关只听 `127.0.0.1`，不对外暴露。

## 4. opencode.json 改指本地 + 消灭悬空引用

- `opencode-go.baseURL → http://127.0.0.1:18080/zen/go/v1`
- `tokyo.baseURL → http://127.0.0.1:18081/zen/v1`
- `small_model` 别指向已删的 provider（曾悬空指向 `modelarts/...`，provider 列表只剩
  `opencode-go/tokyo`，启动不报错但调用必炸；改成现存 provider 下的模型）。
- 同一 key 可同时调个人中转与官方 zen；`free` 档模型只能本体调（带 session 头），
  curl 直测必拒是设计如此，不是链路问题（见 §5）。

## 5. 验证：证明两路真分离（三板斧）

1. **双口 `models` 都是 200**：`curl http://127.0.0.1:18080/zen/go/v1/models` 与
   `:18081/zen/v1/models`。注意**带上路径前缀**，漏了会 404，别误判成链路故障；
   auth 取对字段，错了会 401。
2. **vm-proxy 开 xray access log，按时间戳对照 inbound tag**：
   `[proxy]`（=443 默认入口）即 Go 固定路，`[eg-in-1/2/3]`（=8443/9443/10443）
   即 Zen 轮换池。`loglevel` 保持 `warning`，只加 `access` 文件即可。
3. **终极手段：vm-proxy 上 tcpdump 抓 SYN + 时间戳对齐**：发 Go/Zen 请求并记录时间，
   看新 SYN 的源 IP/端口。曾用此法发现 Go 实际走 v6 出口（见 §6.5）。

正常但别误判的返回：

| 现象 | 定性 |
|------|------|
| Go 经网关 `400 MissingSessionID` | 设计如此，Go/free tier 必须本体带 session 头调 |
| Zen `401 Insufficient balance` | Zen 账户欠费，直调官方也一样，先查账单 |
| `models.dev` 经代理巨慢（滴灌式 tarpit，首包后 ~22KB/s） | 对端反爬/限流，opencode 10s 超时先 `/compact` 或新会话，别怀疑链路 |
| `api.ip.sb` 403 | 它自家反爬，换 `ipinfo.io/ip` 验出口 |

## 6. 踩坑记录

- **6.1 `interval > idle_timeout` 直接 FATAL**：urltest 的 `interval(1h)` 大于
  `idle_timeout(30m)` 时 sing-box 起不来，`Restarting` 死循环。修法：`idle_timeout=2h`。
- **6.2 容器内 `listen 127.0.0.1` 配端口映射必死**：docker-proxy 转发不进容器回路，
  容器内一律 `listen 0.0.0.0`，对外收口靠 compose 的 `127.0.0.1:<port>:<port>`。
- **6.3 `serve --hostname 0.0.0.0` + 代理 env**：goal 插件回调
  `http://0.0.0.0:4006/log` 会进代理黑洞（`ConnectionRefused`），`no_proxy` 必须加
  `0.0.0.0`。修好后 `/log` 回 `400 Missing key` 即正常（不再是连接层错误）。
- **6.4 三层 ssh 引号嵌套传文件必炸**：`base64 invalid input` 基本都是引号吃掉换行/拼接符，
  别拼 base64，分段直传 + 落盘 `od` 验字节数。
- **6.5 Happy Eyeballs 会偷走"固定"**：`opencode.ai` 有 AAAA 且排前面，
  xray 默认出站（freedom）会优先选 v6。本环境歪打正着（v6 全机唯一地址，等于固定了），
  但换环境必须先 `dig AAAA` 确认——别默认"从 443 出去就是主 IPv4"。
  如需强制 v4 才加 `domainStrategy UseIPv4`，默认不动。
- **6.6 验"通没通"不够，要验"从哪出去"**：每次改完对照 access log tag 或 tcpdump 源 IP，
  否则分流没生效（两路同走一个池）根本发现不了。
- **6.7 NO_PROXY 铁律**：本地回路 + 自建中转域名进 NO_PROXY（中转靠直连最快，
  被代理 env 绕去东京再回来纯属加戏）；**被地区锁的域名绝不进 NO_PROXY**。

## 7. 运维与回滚

- 改前备份三件套：`config.json`（sing-box）、`opencode.json`、`config.json`（xray），后缀日期。
- 回滚 = provider `baseURL` 指回官方域名 + 停掉网关 systemd，sing-box 原 2080 入口全程不动。
- TODO：`access.log` 无 logrotate 会涨（共享机器上 log 可见他人目标域名，自行评估）；
  npm 认 env 代理能用但慢（`npm ping` ~1.7s），暂不动。
