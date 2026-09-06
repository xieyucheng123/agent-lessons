# Linux 东京代理最佳实践（sing-box + Xray，换电脑开荒用）

> 目标：一台东京小鸡做出口，本地 sing-box 做 127.0.0.1:2080 混合入口；
> 国外走代理、国内直连；GitHub/opencode.ai 等被墙域名强制走代理。
> 本仓所有敏感值均为占位符（`<...>`），真值只存在本地 `~/.ssh/`、OBS 私有桶，不进 git。

## 0. 架构

```
浏览器/CLI/git ──► 127.0.0.1:2080 (sing-box, docker 常驻)
                        │ rule: opencode.ai → proxy
                        │ rule: geosite-cn/geoip-cn → direct
                        │ final → proxy
                        ▼
                   <VM_PROXY_IP>:443 (Xray VLESS+REALITY) ──► 出海
```

## 1. 服务端：OCI 东京小鸡 + Xray（VLESS + REALITY + xtls-rprx-vision）

- 端口 `443/tcp`，SNI 伪装 `www.apple.com`（不用真实域名，不用证书）。
- 客户端要素：`<UUID>` / `flow=xtls-rprx-vision` / `pbk=<PUBLIC_KEY>` / `sid=<SHORT_ID>` / `fp=chrome`。
- VLESS 链接模板（真值本地拼，不存仓）：
  `vless://<UUID>@<VM_PROXY_IP>:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.apple.com&fp=chrome&pbk=<PUBLIC_KEY>&sid=<SHORT_ID>&type=tcp&headerType=none#vm-proxy`
- 手机端：v2rayNG / Streisand 扫二维码即可。

## 2. 本地：sing-box（docker，常驻自启）

`~/docker/sing-box/docker-compose.yml`：

```yaml
services:
  sing-box:
    image: ghcr.io/sagernet/sing-box:latest
    container_name: sing-box
    restart: always
    ports:
      - "127.0.0.1:2080:2080"
    volumes:
      - ./config.json:/etc/sing-box/config.json:ro
    command: -D /var/lib/sing-box -C /etc/sing-box run
```

`~/docker/sing-box/config.json` 要点（`chmod 600`）：

- `inbounds`: `type: mixed`, `listen: 127.0.0.1`, `port: 2080`
- `outbounds`: `direct` + `proxy`（vless，出站 `server=<VM_PROXY_IP> port=443`，reality 参
数如第 1 节，`flow` 必须填）
- `route.rules` 顺序不能错：
  1. `domain_suffix: [opencode.ai]` → `proxy`（地区锁的域名钉死走代理）
  2. `rule_set: [geosite-cn, geoip-cn]` → `direct`
  3. `final: proxy`
- `rule_set` 用 remote 二进制（`download_detour: proxy`），URL 走
  `SagerNet/sing-geosite` 与 `sing-geoip` 的 `rule-set/...cn.srs`。
- **原则**：个人中转域名（如自己的 relay）不要加规则，靠 geoip-cn 直连；
  只有被地区锁的域名才值得一条钉死规则。

## 3. 系统代理 + 一键开关（`~/.bashrc`）

```bash
proxy() {
  case "$1" in
    on)
      export http_proxy="http://127.0.0.1:2080"
      export https_proxy="http://127.0.0.1:2080"
      export HTTP_PROXY="http://127.0.0.1:2080"
      export HTTPS_PROXY="http://127.0.0.1:2080"
      export no_proxy="localhost,127.0.0.1,::1,<SELF_RELAY_DOMAIN>"
      export NO_PROXY="localhost,127.0.0.1,::1,<SELF_RELAY_DOMAIN>"
      gsettings set org.gnome.system.proxy mode 'manual'
      ;;
    off)
      unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
      export no_proxy="localhost,127.0.0.1,::1,<SELF_RELAY_DOMAIN>"
      export NO_PROXY="localhost,127.0.0.1,::1,<SELF_RELAY_DOMAIN>"
      gsettings set org.gnome.system.proxy mode 'none'
      ;;
  esac
}
```

- gsettings 三处（http/https/socks）都指 `127.0.0.1:2080`，`~/.profile` 里默认 export。
- `NO_PROXY` 清单原则：**本地回路 + 自建中转域名**（中转靠 geoip 直连最快，
  被代理 env 绕去东京再回来纯属加戏）；**被地区锁的域名绝不进 NO_PROXY**。
- `proxy status` 自查：env + gsettings + `docker inspect sing-box` 三样。

## 4. git / GitHub（三块，永久生效，不受 proxy on/off 影响）

```bash
# 4.1 HTTPS：按域名精准走代理（git 不支持通配符，逐个配）
for h in github.com gist.github.com api.github.com codeload.github.com \
         raw.githubusercontent.com objects.githubusercontent.com; do
  git config --global http.https://$h.proxy 'http://127.0.0.1:2080'
done

# 4.2 别留过期证书配置（曾有人 http.sslcainfo 指向已删文件导致全仓 128）
git config --global --unset http.sslcainfo
```

```ssh
# 4.3 SSH：~/.ssh/config（SSH 不认 env，靠 ProxyCommand 进 CONNECT 隧道；443 防 22 被墙）
Host github.com
  HostName ssh.github.com
  Port 443
  User git
  ProxyCommand nc -X connect -x 127.0.0.1:2080 %h %p
```

```bash
# 4.4 gh CLI：认 env，proxy on 后直接用
proxy on && gh api user --jq .login
git ls-remote https://github.com/<OWNER>/<REPO>.git HEAD
ssh -T git@github.com
```

回退：`git config --global --unset-all http.https://github.com.proxy` + 删 ssh 段。

## 5. opencode 注意事项

- opencode 认 `HTTP(S)_PROXY`，但必须 `NO_PROXY=localhost,127.0.0.1`
  防 TUI 本地回路；TUI 进程启动时继承 env，**改代理后必须重启 TUI**。
- opencode 超时约 10s：大上下文（100K+ tokens）推理本身 30s+，
  超时先 `/compact` 或新会话，别先怀疑链路。
- 同一 key 可同时调个人中转与官方 zen（`https://opencode.ai/zen/v1`）；
  官方匿名可调但易 429，假 key 直接 401。

## 6. 自建中转排障手册（nginx 反代官方的场景）

- 对照三连：真直连（`env -u ... curl`）/ 走代理（`-x 127.0.0.1:2080`）/
  小包大包（11 tokens vs 500KB+），先定性是链路段还是推理段。
- 分阶段计时：`-w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} 首包:%{time_starttransfer}"`，
  卡在哪段一目了然。
- 源站没 IPv6 出口时，nginx 上游（Cloudflare 双栈）会先撞 v6 墙：
  `resolver ... ipv6=off` + `proxy_connect_timeout 5s`（别用 30s，会吃掉客户端超时预算）。
- OCI 开 IPv6 六步：VCN 加段 → 子网切 /64 → 路由表 `::/0` 走 IGW
  （`route-table update --force`，记得带上原有 v4 规则）→ 安全表出站放 `::/0`
  → 网卡 `oci network ipv6 create` → 系统 SLAAC 自动生效（`ip -6 addr` 验证）。
- `--rm` 容器**禁用 `podman restart`**（会删容器）；走 systemd 单元
  `systemctl restart <name>.service`。
- 个人中转 + 公有 WAF：先查自家 nginx 日志（`upstream timed out` / `Network unreachable`），
  再怀疑 WAF（Bot 对抗会 tarpit 高频测试 IP，自家出口 IP 去 WAF 控制台加白）。
- WAF 回源段会变：源站收紧只做 nginx 层 allow（回滚快），别在云安全组硬收紧。

## 7. 新电脑开荒顺序

1. 装 docker + 拉 sing-box 镜像，起 `~/docker/sing-box`（第 2 节）。
2. 验收：`curl -x 127.0.0.1:2080 ip.sb` 显示东京 IP；`curl myip.ipip.net` 显示本地。
3. 配 `proxy()` + gsettings + `~/.profile`（第 3 节）。
4. 配 git/ssh/gh（第 4 节），三条验证命令全过。
5. 按需配 opencode（第 5 节）。
