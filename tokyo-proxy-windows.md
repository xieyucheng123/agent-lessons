# Windows 东京代理最佳实践（sing-box 原生 exe + 计划任务，配套 Linux 版）

> Linux 版见 [tokyo-proxy-best-practice.md](tokyo-proxy-best-practice.md)。
> 本篇记录 Windows 机器开荒的实际踩坑：winget 装不上、curl 吊销检查报错、
> 计划任务替代 docker restart:always、双代理体系（env vs WinINET）。
> 所有敏感值均为占位符（`<...>`），真值只在本地，不进 git。

## 0. 架构

```
CLI/git/gh/桌面应用 ──► 127.0.0.1:2080 (sing-box, 计划任务常驻)
                            │ rule: opencode.ai → proxy
                            │ rule: geosite-cn/geoip-cn → direct
                            │ final → proxy
                            ▼
                       <VM_PROXY_IP>:443 (Xray VLESS+REALITY) ──► 出海
```

与 Linux 版唯一区别：本地用原生 exe + 计划任务，不用 docker。

## 1. 先诊断旧环境，别急着装新的

Windows 常驻着一个机场客户端改的 User 级 env（如 iKuuu 的 7890），它坏了会连累一切：

```powershell
# 查 User 级现有代理 env
[Environment]::GetEnvironmentVariable('HTTPS_PROXY','User')

# 定性死活：CONNECT 200 但 TLS 秒断 = 节点坏死，换出口别修它
curl.exe -sv -x http://127.0.0.1:7890 https://www.google.com -o NUL --max-time 10
```

- 现象特征：`CONNECT` 返回 200 后 TLS 握手立即失败，被墙/正常站全 `000`，
  只有国内站能过 → 机场节点死了。此时**别把希望寄托在修它上**，直接上自己的 sing-box。
- `opencode.ai` 直连本身也被墙（000），所以必须先有可用出口才能谈官方 API。

## 2. 装 sing-box：winget 大概率失败，用镜像直连

winget 装会连环炸：msstore 源报错 + GitHub release 下载失败
（`InternetOpenUrl 0x80072f19`，WinINET 走的旧系统代理已死）。别缠斗，镜像直连：

```powershell
$ver = '1.14.0'
curl.exe -L --ssl-no-revoke --noproxy "*" `
  -o "$env:TEMP\sing-box.zip" `
  "https://gh-proxy.com/https://github.com/SagerNet/sing-box/releases/download/v$ver/sing-box-$ver-windows-amd64.zip"
Expand-Archive "$env:TEMP\sing-box.zip" -DestinationPath "$env:USERPROFILE\sing-box" -Force
```

- **`--ssl-no-revoke` 必加**：Windows curl 用 Schannel，吊销服务器不可达直接
  `CRYPT_E_REVOCATION_OFFLINE`，没有这根签什么都下不了（鸡生蛋问题）。
- **`--noproxy "*"` 必加**：下载镜像时绕开已死的旧代理 env。

装完 `sing-box check -c config.json` 再启动，别裸跑。

## 3. config.json：与 Linux 版同构

规则顺序照抄 Linux 版第 2 节（opencode.ai 钉死 → geosite/geoip-cn 直连 → final proxy），
Windows 特有的差异点：

- `dns.servers`：`223.5.5.5`（`detour: direct`）+ `8.8.8.8`（`detour: proxy`），
  别让 DNS 查询自己先被墙。
- `cache_file` 开启（Windows 没有 docker volume，cache 就落安装目录）。
- 中转域名（自己的 relay、内网 MaaS 端点）**不加规则**，靠 NO_PROXY 解决（见第 5 节）。

## 4. 常驻：计划任务替代 docker restart:always

```powershell
$action = New-ScheduledTaskAction -Execute "$env:USERPROFILE\sing-box\bin\sing-box.exe" `
  -Argument "run -c $env:USERPROFILE\sing-box\config.json" `
  -WorkingDirectory "$env:USERPROFILE\sing-box"
$tBoot  = New-ScheduledTaskTrigger -AtStartup; $tBoot.Delay = 'PT30S'
$tLogon = New-ScheduledTaskTrigger -AtLogon -User "$env:COMPUTERNAME\$env:USERNAME"
$principal = New-ScheduledTaskPrincipal -UserId "$env:COMPUTERNAME\$env:USERNAME" -LogonType S4U -RunLevel Highest
$settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit ([TimeSpan]::Zero) `
  -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1) `
  -StartWhenAvailable -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName 'sing-box' -Action $action -Trigger $tBoot,$tLogon `
  -Principal $principal -Settings $settings -Force
```

三个关键点：

- **`-LogonType S4U`**：免存密码 + 控制台程序无黑窗弹出（普通"仅登录时运行"会闪一个控制台）。
- **`-ExecutionTimeLimit ([TimeSpan]::Zero)` 必设**：默认 72 小时时限会半夜杀掉常驻进程。
- `AtStartup`（延迟 30s）+ `AtLogon` 双触发：开机即有代理，登录后兜底。
  改完配置：`Stop-Process -Name sing-box -Force; Start-ScheduledTask sing-box`。

## 5. 环境变量：User 级 + NO_PROXY 清单

```powershell
$proxy = 'http://127.0.0.1:2080'
foreach ($n in 'HTTP_PROXY','HTTPS_PROXY','http_proxy','https_proxy') {
  [Environment]::SetEnvironmentVariable($n, $proxy, 'User')
}
$noProxy = '127.0.0.1,localhost,::1,192.168.0.0/16,10.0.0.0/8,*.local,<SELF_RELAY_DOMAIN>,<LAN_MAAS_ENDPOINT>'
foreach ($n in 'NO_PROXY','no_proxy') {
  [Environment]::SetEnvironmentVariable($n, $noProxy, 'User')
}
```

- **大写小写各设一份**：不同程序读不同（Node 读小写为主，Go/.NET 读大写）。
- `.NET SetEnvironmentVariable` 会广播 `WM_SETTINGCHANGE`，Explorer 能感知，
  但**已运行的进程不会**——桌面版 opencode（TUI 同理）必须完全退出（含托盘）再开，
  它才拿到新 env。
- NO_PROXY 原则同 Linux 版：本地回路 + 局域网端点（内网 litellm、云厂商 MaaS）+
  自建中转域名；被地区锁的域名绝不进 NO_PROXY。

## 6. Windows 双代理体系：env ≠ 浏览器

Linux 上 gsettings 一改全家生效；Windows 是**两套独立系统**：

| 体系 | 覆盖 | 配置位置 |
|------|------|----------|
| env 变量 | CLI、git、gh、opencode 桌面版 | User 环境变量（第 5 节） |
| WinINET 系统代理 | 浏览器、winget 等 GUI | Internet 选项（设置里手动改） |

- 改 User env **不会**动浏览器——浏览器继续走机场客户端的系统代理（或裸连）。
  这是特性不是 bug：机场死了也不影响 CLI 工作。
- winget/Edge 等吃 WinINET 的程序想要新出口，需在"设置 → 网络 → 代理"手动改
  或用机场客户端的开关节。

## 7. 验收清单

```powershell
# 走东京：被墙站应 200
curl.exe -s -o NUL -w "%{http_code} %{time_total}s" -x http://127.0.0.1:2080 --ssl-no-revoke https://opencode.ai
# 直连规则生效：国内站 0.1s 级（若也 3s+ 说明规则没生效）
curl.exe -s -o NUL -w "%{http_code} %{time_total}s" -x http://127.0.0.1:2080 --ssl-no-revoke https://www.baidu.com
# 任务托管：State 应为 Running
(Get-ScheduledTask -TaskName 'sing-box').State
```

- `api.ip.sb` 返回 403 是它自家反爬，别误判成链路故障；换 `ipinfo.io/ip` 验出口 IP。
- git/gh 部分直接复用 Linux 版第 4 节（`http.<url>.proxy` 逐域名配、ssh 走
  `connect -x 127.0.0.1:2080`，Windows 自带 `connect.exe` 可替代 `nc`）。

## 8. Windows 开荒顺序

1. 诊断旧代理 env 是否坏死（第 1 节）。
2. 镜像直连装 sing-box（第 2 节），check 后起任务（第 3、4 节）。
3. 设 User 级 env + NO_PROXY（第 5 节）。
4. 验收三连（第 7 节），重启桌面应用。
5. 浏览器要不要切，按需决定（第 6 节）。
