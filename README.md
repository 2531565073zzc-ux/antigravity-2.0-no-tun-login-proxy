# Antigravity 2.0 No-TUN Login Proxy Fix

> 针对 Antigravity 2.0 / 2.0.1 的无 TUN 登录代理修复方案。  
> 重点解决：不开 TUN 模式时，Antigravity 登录页、Agent 初始化、`daily-cloudcode-pa.googleapis.com` 请求超时的问题。

作者 GitHub：

[https://github.com/2531565073zzc-ux](https://github.com/2531565073zzc-ux)

参考工具项目：

[yuaotian/antigravity-proxy](https://github.com/yuaotian/antigravity-proxy)


本仓库已经提供待配置文件：

```text
version.dll
config.json
```

用户只需要从本仓库拿到这两个文件（Release 页面下载文件），按本文步骤复制到 Antigravity 安装目录即可。

---

## 文档导航

- [这个方案解决什么](#这个方案解决什么)
- [适合哪些人使用](#适合哪些人使用)
- [Antigravity 2.0 的关键变化](#antigravity-20-的关键变化)
- [最核心的一行修复](#最核心的一行修复)
- [本仓库提供的文件](#本仓库提供的文件)
- [快速部署流程](#快速部署流程)
- [推荐配置文件](#推荐配置文件)
- [如何确认真的修好了](#如何确认真的修好了)
- [不开 TUN 模式时的注意点](#不开-tun-模式时的注意点)
- [常见失败原因](#常见失败原因)
- [日志怎么看](#日志怎么看)
- [进阶配置建议](#进阶配置建议)
- [安全与公开发布提醒](#安全与公开发布提醒)
- [声明与致谢](#声明与致谢)

---

## 这个方案解决什么

很多用户在使用 Antigravity 2.0 时，会遇到登录页面卡住或 Agent 无法启动的问题。

典型页面提示：

```text
Authentication Required

To start using the agent, please sign in with your Google account.

Post "https://daily-cloudcode-pa.googleapis.com/v1internal:loadCodeAssist":
context deadline exceeded
```

这个报错看起来像是账号没有登录成功，但实际经常是网络链路没有走代理。

尤其是在以下使用方式中更容易出现：

- Clash / Mihomo / Clash Verge 没有开启 TUN
- V2RayN 只开了本地 SOCKS5 / HTTP 端口
- Windows 系统代理设置了，但 Antigravity 子进程没有继承
- 浏览器能访问 Google，但 Antigravity Agent 仍然报错
- `Antigravity.exe` 看似走代理了，但 `language_server.exe` 没有走代理

本方案解决的就是这条链路：

```text
Antigravity 2.0
  -> language_server.exe
    -> daily-cloudcode-pa.googleapis.com
      -> 通过本地代理访问
```

换句话说，这不是单纯的“登录按钮坏了”，而是让 Antigravity 2.0 的关键服务进程正确进入代理链路。

---

## 适合哪些人使用

如果你符合下面任意一种情况，这份文档大概率适合你：

- 你不想开启 TUN 模式
- 你只想让 Antigravity 走代理，不想全局接管网络
- 你已经部署了 `antigravity-proxy`，但仍然无法登录
- 你看到 `daily-cloudcode-pa.googleapis.com` 超时
- 你使用的是 Antigravity 2.0 / 2.0.1
- 你的代理端口是 `7897`、`7890`、`7891`、`10808` 等本地端口
- 你希望保留 `filtered` 精准注入模式，而不是注入所有子进程

不适合的情况：

- 你的本地代理端口本身不通
- 你的代理出口 IP 已经被 Google 服务链路限制
- 你的问题是账号权限、地区风控、组织策略或服务不可用
- 你已经开启 TUN 并且所有进程都能稳定走代理

---

## Antigravity 2.0 的关键变化

Antigravity 2.0 不只是一个 `Antigravity.exe`。

它会启动一个非常关键的语言服务进程：

```text
resources\bin\language_server.exe
```

从 Antigravity 主日志中可以看到类似内容：

```text
Spawning:
C:\Users\<用户名>\AppData\Local\Programs\antigravity\resources\bin\language_server.exe
--standalone
--api_server_url https://generativelanguage.googleapis.com
--cloud_code_endpoint https://daily-cloudcode-pa.googleapis.com
```

这里有两个重点：

```text
language_server.exe
```

和：

```text
daily-cloudcode-pa.googleapis.com
```

也就是说，Antigravity 2.0 的登录、Agent 初始化、模型列表、Code Assist 等能力，并不完全由 `Antigravity.exe` 直接完成。

实际链路更接近：

```text
Antigravity.exe
  -> 启动 language_server.exe
  -> language_server.exe 请求 Google / Cloud Code 服务
  -> 返回登录状态、模型能力、Agent 状态
```

如果只让 `Antigravity.exe` 走代理，而 `language_server.exe` 被跳过，那么页面仍然可能显示：

```text
context deadline exceeded
```

这就是本文档与普通端口配置教程最大的区别。

---

## 最核心的一行修复

在 `config.json` 的 `target_processes` 中加入：

```json
"language_server.exe"
```

推荐写法：

```json
"target_processes": [
  "language_server_windows",
  "language_server.exe",
  "Antigravity.exe",
  "node.exe"
]
```

为什么要保留 `language_server_windows`？

因为不同版本或不同构建中，语言服务进程名称可能不同。保留它可以兼容旧配置，同时新增 `language_server.exe` 适配 Antigravity 2.0。

如果不加这一项，日志中可能出现：

```text
[跳过] child_injection_mode=filtered 非目标进程: language_server.exe
```

这表示真正发起 `daily-cloudcode-pa.googleapis.com` 请求的进程没有被代理注入。

---

## 本仓库提供的文件

本仓库根目录提供两个核心文件：

```text
version.dll
config.json
```

用途如下：

| 文件 | 作用 |
| --- | --- |
| `version.dll` | 代理注入 DLL，放到 `Antigravity.exe` 同级目录后，会随 Antigravity 启动加载 |
| `config.json` | 代理配置文件，用来指定本地代理端口、代理类型、目标进程和路由策略 |

推荐仓库结构：

```text
your-repo/
├─ README.md
├─ version.dll
└─ config.json
```

用户操作时只需要复制：

```text
version.dll
config.json
```

到 Antigravity 安装目录：

```text
%LOCALAPPDATA%\Programs\antigravity
```

也就是和下面这个文件放在一起：

```text
Antigravity.exe
```

复制完成后的目录应类似：

```text
C:\Users\<用户名>\AppData\Local\Programs\antigravity\Antigravity.exe
C:\Users\<用户名>\AppData\Local\Programs\antigravity\version.dll
C:\Users\<用户名>\AppData\Local\Programs\antigravity\config.json
```

---

## 快速部署流程

以下示例使用本地代理端口：

```text
127.0.0.1:7897
```

如果你的端口不是 `7897`，请替换成自己的实际端口。

### 第一步：确认本地代理端口可用

PowerShell 执行：

```powershell
Test-NetConnection -ComputerName 127.0.0.1 -Port 7897
```

成功时应看到：

```text
TcpTestSucceeded : True
```

如果是 `False`，说明代理软件没有监听该端口，后续配置不会生效。

### 第二步：获取本仓库文件

从本仓库获取下面两个文件：

```text
version.dll
config.json
```

不需要再去其他仓库的 Release 页面下载。

如果你是通过 GitHub 页面下载，可以点击仓库页面的：

```text
Code -> Download ZIP
```

下载并解压后，确认目录中有：

```text
version.dll
config.json
```

如果你使用 git，也可以克隆自己的仓库：

```powershell
git clone https://github.com/2531565073zzc-ux/<你的仓库名>.git
```

然后进入仓库目录，找到：

```text
version.dll
config.json
```

### 第三步：找到 Antigravity 安装目录

默认路径一般是：

```text
%LOCALAPPDATA%\Programs\antigravity
```

完整示例：

```text
C:\Users\<用户名>\AppData\Local\Programs\antigravity
```

目录中应能看到：

```text
Antigravity.exe
resources
locales
```

### 第四步：复制文件

把本仓库根目录中的两个文件复制到 Antigravity 安装目录：

```text
version.dll
config.json
```

复制后应类似：

```text
C:\Users\<用户名>\AppData\Local\Programs\antigravity\Antigravity.exe
C:\Users\<用户名>\AppData\Local\Programs\antigravity\version.dll
C:\Users\<用户名>\AppData\Local\Programs\antigravity\config.json
```

注意：`version.dll` 必须和 `Antigravity.exe` 在同一目录。

### 第五步：修改 config.json

打开：

```text
%LOCALAPPDATA%\Programs\antigravity\config.json
```

确认代理端口：

```json
"proxy": {
  "host": "127.0.0.1",
  "port": 7897,
  "type": "socks5"
}
```

如果你的代理端口不是 `7897`，只需要修改 `port`。

例如本地代理是 `7890`：

```json
"proxy": {
  "host": "127.0.0.1",
  "port": 7890,
  "type": "socks5"
}
```

如果你的代理端口是 `10808`：

```json
"proxy": {
  "host": "127.0.0.1",
  "port": 10808,
  "type": "socks5"
}
```

确认目标进程：

```json
"target_processes": [
  "language_server_windows",
  "language_server.exe",
  "Antigravity.exe",
  "node.exe"
]
```

### 第六步：完全重启 Antigravity

关闭 Antigravity 后，建议确认这些进程都已经退出：

```text
Antigravity.exe
language_server.exe
node.exe
```

然后重新打开：

```text
%LOCALAPPDATA%\Programs\antigravity\Antigravity.exe
```

旧的 `language_server.exe` 不会自动读取新配置，所以必须重启。

### 第七步：点击 Sign In 并等待加载

重新打开 Antigravity 后，进入登录页面，点击：

```text
Sign In
```

如果页面仍然显示旧错误，请先完全退出 Antigravity，再重新打开一次。

重点不是反复点击按钮，而是确认日志中已经出现：

```text
[成功] 已注入目标进程: language_server.exe
SOCKS5: 隧道建立成功, 目标=daily-cloudcode-pa.googleapis.com:443
```

---

## 推荐配置文件

下面是针对 Antigravity 2.0、不开 TUN、本地 SOCKS5 端口为 `7897` 的推荐关键配置：

```json
{
  "proxy": {
    "host": "127.0.0.1",
    "port": 7897,
    "type": "socks5"
  },
  "child_injection": true,
  "child_injection_mode": "filtered",
  "target_processes": [
    "language_server_windows",
    "language_server.exe",
    "Antigravity.exe",
    "node.exe"
  ],
  "fake_ip": {
    "enabled": true,
    "cidr": "198.18.0.0/15"
  },
  "timeout": {
    "connect": 5000,
    "send": 5000,
    "recv": 5000
  },
  "log_level": "info",
  "traffic_logging": false,
  "proxy_rules": {
    "allowed_ports": [
      80,
      443
    ],
    "dns_mode": "direct",
    "ipv6_mode": "proxy",
    "udp_mode": "block",
    "udp_fallback": "block",
    "routing": {
      "enabled": true,
      "priority_mode": "order",
      "default_action": "proxy",
      "use_default_private": true,
      "rules": []
    }
  }
}
```

端口按你的代理软件实际配置调整。

常见示例：

| 代理客户端 | 常见 SOCKS5 端口 | 常见 HTTP / Mixed 端口 |
| --- | --- | --- |
| Clash / Mihomo / Clash Verge | 7891 | 7890 |
| Clash mixed-port | 7890 | 7890 |
| V2RayN | 10808 | 10809 |
| Shadowsocks | 1080 | 视客户端配置而定 |
| 自定义端口 | 7897 | 7897 |

如果你的端口同时支持 SOCKS5 和 HTTP，推荐优先使用：

```json
"type": "socks5"
```

---

## 如何确认真的修好了

修复不是看页面是否立刻刷新，而是先看日志链路是否正确。

日志位置：

```text
%LOCALAPPDATA%\Programs\antigravity\logs\proxy-YYYYMMDD.log
```

### 1. 确认配置读取成功

应该看到：

```text
配置: proxy=127.0.0.1:7897 type=socks5
```

以及：

```text
配置加载成功
```

### 2. 确认目标进程数量变为 4

如果你加入了 `language_server.exe`，日志中应看到类似：

```text
已加载目标进程列表: 共 4 项
```

如果仍然是 3 项，说明安装目录中的 `config.json` 没有改对。

### 3. 确认 language_server.exe 已注入

正确日志：

```text
[成功] 已注入目标进程: language_server.exe
```

错误日志：

```text
[跳过] child_injection_mode=filtered 非目标进程: language_server.exe
```

如果看到“跳过”，说明 `target_processes` 仍然没有匹配。

### 4. 确认 daily-cloudcode 走代理

正确日志：

```text
ConnectEx 正重定向 daily-cloudcode-pa.googleapis.com:443 到代理
```

以及：

```text
SOCKS5: 隧道建立成功, 目标=daily-cloudcode-pa.googleapis.com:443
```

看到这两类日志，才说明 Antigravity 2.0 的登录 / Agent 关键请求已经真正通过代理。

### 5. 确认语言服务初始化

Antigravity 自己的语言服务日志在：

```text
%APPDATA%\Antigravity\logs\language_server.log
```

如果看到：

```text
initialized server successfully
```

并且不再持续刷：

```text
daily-cloudcode-pa.googleapis.com ... context deadline exceeded
```

说明链路基本正常。

---

## 不开 TUN 模式时的注意点

不开 TUN 模式时，代理客户端通常只提供本地端口，例如：

```text
127.0.0.1:7897
```

这时浏览器能走代理，并不代表所有 Windows 程序都会自动走代理。

原因包括：

- 有些程序不读取系统代理
- 有些子进程不继承代理环境变量
- 有些网络库直接调用底层 Winsock
- 有些请求由后台服务进程发起
- Antigravity 2.0 的关键请求在 `language_server.exe` 中

所以 No-TUN 模式下的关键不是“浏览器能不能打开 Google”，而是：

```text
language_server.exe 是否被代理注入
```

这也是本文档的核心。

---

## 常见失败原因

### 1. 只改了端口，没有加 language_server.exe

错误配置：

```json
"target_processes": [
  "language_server_windows",
  "Antigravity.exe",
  "node.exe"
]
```

结果：

```text
language_server.exe 被跳过
daily-cloudcode-pa.googleapis.com 请求超时
```

### 2. 只改了仓库里的 config.json，但没有覆盖安装目录

如果你只是修改了本仓库里的：

```text
config.json
```

但没有把它复制到 Antigravity 安装目录，Antigravity 仍然读取不到新配置。

Antigravity 实际读取的是：

```text
%LOCALAPPDATA%\Programs\antigravity\config.json
```

请确认安装目录里的 `config.json` 已经被本仓库的配置文件覆盖。

### 3. Antigravity 没有完全重启

配置修改后，已经运行中的 `language_server.exe` 不会自动重新注入。

建议完全退出后重启。

### 4. 代理端口协议写错

如果你的端口只支持 HTTP，却写成：

```json
"type": "socks5"
```

可能导致握手失败。

如果不确定端口类型，请查看代理客户端设置。

### 5. 代理出口 IP 被限制

如果已经看到 SOCKS5 隧道成功，但仍然出现：

```text
User location is not supported for the API use
```

这通常是出口 IP 问题。

建议尝试：

- 换节点
- 换地区
- 使用普通 ISP / 住宅出口
- 避免数据中心 IP
- 检查 Google 账号服务权限

---

## 日志怎么看

建议按下面顺序看日志。

### 1. antigravity-proxy 日志

位置：

```text
%LOCALAPPDATA%\Programs\antigravity\logs\proxy-YYYYMMDD.log
```

重点搜索：

```text
proxy=127.0.0.1
language_server.exe
daily-cloudcode-pa.googleapis.com
SOCKS5
跳过
成功
```

### 2. Antigravity 主日志

位置：

```text
%APPDATA%\Antigravity\logs\main.log
```

重点搜索：

```text
Spawning:
```

用来确认 Antigravity 实际启动的语言服务路径。

### 3. Antigravity 语言服务日志

位置：

```text
%APPDATA%\Antigravity\logs\language_server.log
```

重点搜索：

```text
daily-cloudcode-pa.googleapis.com
loadCodeAssist
fetchAvailableModels
context deadline exceeded
User location is not supported
initialized server successfully
```

---

## 进阶配置建议

### 保持 filtered 模式

推荐：

```json
"child_injection_mode": "filtered"
```

这样只注入目标进程，更稳定，也更容易看日志。

### 不建议默认 inherit

虽然可以改成：

```json
"child_injection_mode": "inherit"
```

但它会尝试注入更多子进程。

缺点：

- 日志更杂
- 影响范围更大
- 排障不够精确
- 可能影响无关进程

除非你确认还有其他必要进程被跳过，否则更推荐补充 `target_processes`。

### 可以临时打开 debug

排查时可以改成：

```json
"log_level": "debug"
```

排查完成后建议恢复：

```json
"log_level": "info"
```

### 不建议长期打开 traffic_logging

可以临时使用：

```json
"traffic_logging": true
```

但发布日志前一定要检查敏感信息。

---

## 安全与公开发布提醒

如果你准备把排障过程发到 GitHub，不建议直接上传完整日志。

日志中可能包含：

- access token
- OAuth token
- 用户名
- 本地路径
- 账号相关信息
- 请求 Trace
- 其他认证信息

建议只保留这些无敏感字段：

```text
进程名
目标域名
端口
是否注入成功
是否建立 SOCKS5 隧道
错误类型
```

例如可以公开：

```text
[成功] 已注入目标进程: language_server.exe
SOCKS5: 隧道建立成功, 目标=daily-cloudcode-pa.googleapis.com:443
```

不建议公开：

```text
access_token=...
Authorization: Bearer ...
本机完整用户名路径
```

---

## 声明与致谢

本方案基于：

[yuaotian/antigravity-proxy](https://github.com/yuaotian/antigravity-proxy)

该项目提供了 `version.dll` 注入和代理重定向能力。

本文档的重点是针对 Antigravity 2.0 / 2.0.1 的新现象进行补充说明：

```text
Antigravity 2.0 的关键请求由 language_server.exe 发起，
No-TUN 模式下必须确保 language_server.exe 被注入并走代理。
```

如果你发布自己的仓库，建议说明：

- 本仓库提供的是 Antigravity 2.0 No-TUN 登录问题的配置文件和操作说明
- 核心代理注入能力来自参考工具项目
- 使用原项目文件时应遵守原项目许可证
- 不要删除原项目作者信息和许可证声明

---

## 最短总结

不开 TUN 模式时，Antigravity 2.0 登录失败不一定是代理端口错了。

真正需要确认的是：

```text
language_server.exe 是否也走了代理
```

关键配置：

```json
"target_processes": [
  "language_server_windows",
  "language_server.exe",
  "Antigravity.exe",
  "node.exe"
]
```

关键成功日志：

```text
[成功] 已注入目标进程: language_server.exe
SOCKS5: 隧道建立成功, 目标=daily-cloudcode-pa.googleapis.com:443
```

看到这两类日志，才说明 Antigravity 2.0 的登录和 Agent 关键链路真正进入了本地代理。
