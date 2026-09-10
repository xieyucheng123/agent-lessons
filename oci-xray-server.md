# OCI 机器搭建 Xray 多出口代理服务端（VLESS + REALITY，从零开始）

> 目标：在 OCI 東京机器上起 Xray，4 个端口对应 4 个出口——443 默认出口 +
> 8443/9443/10443 各绑一个固定私网 IP（给 [Go 固定 / Zen 轮换分流](proxy-go-fixed-zen-rotate.md) 用）。
> 单端口够用的场景看 [tokyo-proxy-best-practice.md](tokyo-proxy-best-practice.md) §1。
> 所有敏感值均为占位符（`<...>`），真值只在私仓 `agent-secrets/proxy/tokyo.env`，不进 git。

## 0. 架构

```
客户端 ──► <VM_PROXY_HOST>:443              (无 tag, 默认出站) ──► 主出口
         ──► <VM_PROXY_HOST>:8443 (eg-in-1) ──► eg-out-1 (sendThrough <EGRESS_IP_1>)
         ──► <VM_PROXY_HOST>:9443 (eg-in-2) ──► eg-out-2 (sendThrough <EGRESS_IP_2>)
         ──► <VM_PROXY_HOST>:10443 (eg-in-3) ──► eg-out-3 (sendThrough <EGRESS_IP_3>)
```

- 同一个 UUID 走 4 个端口，靠 inbound tag 分到不同出口；客户端想固定就连 443，
  想轮换就让 sing-box `urltest` 在 3 个池子里选（见分流篇）。
- 4 个 inbound 的 REALITY 参数完全相同（同一份 `privateKey` + `shortIds`），
  客户端链接只换端口就能切出口。

## 1. OCI 侧：网络与机器

1. VCN + 子网（能直接上网即可）；**安全列表**放行入站 TCP
   `443/8443/9443/10443`（源 `0.0.0.0/0`，要开 v6 再加 `::/0`，IPv6 六步见 Linux 版 §6）。
2. 实例：Ubuntu（LTS 即可）+ 分配公网 IP；给机器绑个域名记为 `<VM_PROXY_HOST>`。
   **客户端一律填域名不要填 IP**：REALITY 不校验拨号地址，效果一样；
   IP 变了只需改 DNS，不用逐台重导节点。
3. 多出口的私网 IP：在控制台给这台机器网卡加**辅助私网 IP**（本例 3 个，
   记为 `<EGRESS_IP_1/2/3>`），系统里 `ip addr` 能看到即生效，不用配网卡文件。
   如需每个私网 IP 对应固定公网出口，再去绑**保留公网 IP**——注意去控制台核对
   私网↔公网映射表，**别假设按顺序对应**（曾在此坑过，映射要逐条点开确认）。

## 2. 装 Xray

官方一键脚本（root 或 sudo）：

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
systemctl enable --now xray
xray version
```

- 脚本自带 `geoip.dat`/`geosite.dat`（落 `/usr/local/share/xray/`），
  配置里 `geosite:cn` / `geoip:cn` 规则依赖它俩，缺了路由里的国内直连会失效。
- 配置文件：`/usr/local/etc/xray/config.json`（`chmod 600`）。

## 3. 生成三样密钥（本机或服务端都行）

```bash
xray uuid            # <UUID>，4 个 inbound 共用一个
xray x25519          # 得 PrivateKey <REALITY_PRIVKEY> / PublicKey <PUBLIC_KEY>
openssl rand -hex 8  # <SHORT_ID>
```

- `<REALITY_PRIVKEY>` 放服务端 `realitySettings.privateKey`；
- `<PUBLIC_KEY>` / `<SHORT_ID>` / `<UUID>` 拼客户端链接（模板见 Linux 版 §1，
  4 个端口只换 `:443` → `:8443/:9443/:10443`）。
- 三样真值当时就记进私仓，别在 shell 历史里裸奔完就忘。

## 4. config.json（全文，占位符版）

结构与现行生产配置同构，4 段 key  AAA 去：inbounds 的 `realitySettings`
（`target: www.apple.com:443`，`serverNames: [www.apple.com]`，不用真实域名、
不用证书）、`sniffing` 开 `http,tls`，outbounds 的 `sendThrough`，
routing 的按 tag 分流 + 广告拦截 + 国内直连：

```jsonc
{
  "log": { "loglevel": "warning", "access": "/var/log/xray/access.log" },
  "inbounds": [
    // 443 默认入口：无 tag，走第一个出站（主出口）
    { "listen": "0.0.0.0", "port": 443, "protocol": "vless",
      "settings": { "clients": [{ "id": "<UUID>", "flow": "xtls-rprx-vision" }], "decryption": "none" },
      "streamSettings": { "network": "raw", "security": "reality",
        "realitySettings": { "show": false, "target": "www.apple.com:443", "xver": 0,
          "serverNames": ["www.apple.com"], "privateKey": "<REALITY_PRIVKEY>",
          "shortIds": ["", "<SHORT_ID>"] } },
      "sniffing": { "enabled": true, "destOverride": ["http", "tls"] } },
    // 8443/9443/10443：结构与 443 完全相同，只差 port + tag（eg-in-1/2/3）
    { "port": 8443, /* ...同上... */ "tag": "eg-in-1" },
    { "port": 9443, /* ...同上... */ "tag": "eg-in-2" },
    { "port": 10443, /* ...同上... */ "tag": "eg-in-3" }
  ],
  "outbounds": [
    { "tag": "proxy", "protocol": "freedom" },            // 443 的默认出站（主出口）
    { "tag": "direct", "protocol": "freedom", "settings": { "domainStrategy": "UseIP" } },
    { "tag": "block", "protocol": "blackhole" },
    { "tag": "eg-out-1", "protocol": "freedom", "sendThrough": "<EGRESS_IP_1>" },
    { "tag": "eg-out-2", "protocol": "freedom", "sendThrough": "<EGRESS_IP_2>" },
    { "tag": "eg-out-3", "protocol": "freedom", "sendThrough": "<EGRESS_IP_3>" }
  ],
  "routing": {
    "domainStrategy": "IPIIfNonMatch",
    "rules": [
      { "type": "field", "inboundTag": ["eg-in-1"], "outboundTag": "eg-out-1" },
      { "type": "field", "inboundTag": ["eg-in-2"], "outboundTag": "eg-out-2" },
      { "type": "field", "inboundTag": ["eg-in-3"], "outboundTag": "eg-out-3" },
      { "type": "field", "outboundTag": "block", "domain": ["geosite:category-ads-all"] },
      { "type": "field", "outboundTag": "direct", "domain": ["geosite:cn"] },
      { "type": "field", "outboundTag": "direct", "ip": ["geoip:cn", "geoip:private"] }
    ]
  }
}
```

注意：

- `shortIds` 首元素留空字符串 `""`（兼容不带 shortId 的客户端），别删。
- `direct` 出站 `domainStrategy: UseIP`：国内域名直接解析 IP 直连，别绕路。
- 改完先备份（`cp config.json config.json.bak-$(date +%Y%m%d)`），再
  `systemctl restart xray`，看 `systemctl is-active xray` + `journalctl -u xray`。

## 5. 本机防火墙

OCI Ubuntu 镜像默认 INPUT 全 ACCEPT 时只加端口规则即可；收紧过的机器补：

```bash
for p in 443 8443 9443 10443; do
  sudo iptables -I INPUT -p tcp --dport $p -j ACCEPT
done
```

安全列表（第 1 节）+ 本机 iptables 两层都要放行，缺一层都是超时（不是拒绝，
现象是 `000`/转圈，别当成密钥配错了）。

## 6. 验收：证明 4 个端口真是 4 个出口

1. 客户端（v2rayNG / sing-box）分别连 4 个端口，访问 `ipinfo.io/ip`，
   应看到 4 个不同出口 IP（443 是主出口，3 个池子各一个）。
2. 服务端对照：`tail -f /var/log/xray/access.log`，按时间戳看 inbound tag
   （无 tag = 443，`eg-in-1/2/3` = 3 个池子）。
3. 终极手段：服务端 `tcpdump` 抓出站 SYN 看源 IP（曾用此法定案 Go 走 v6，见分流篇 §6.5）。
4. 提醒：`opencode.ai` 这类双栈域名，默认出站可能走 Happy Eyeballs 选 v6——
   本环境 v6 全机唯一反而等于固定了；换环境先 `dig AAAA` 确认，别默认"443 出去就是主 IPv4"。
   如需强制 v4 才加 `domainStrategy: UseIPv4`，默认不动。

## 7. 日常运维

- 只开 access log（`loglevel` 保持 `warning`，不放大）：`log` 段三行就够，
  排障时看 tag，连通性看客户端。
- `access.log` 无 logrotate 会涨，共享机器上 log 可见他人目标域名——自行评估，
  要转就加 `logrotate`。
- 443 端口被扫是常态：REALITY 没密钥直接 RST，不用理；真被重点关照再换端口+改 DNS。
