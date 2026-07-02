---
name: x-ui-deploy
description: >
  Deploy a working VLESS + XHTTP + TLS + Cloudflare CDN VPN service on a fresh VPS
  using the 3X-UI panel. Goal: get a working proxy node up as quickly as possible.
  Use this skill when the user mentions: deploying a VPN server, setting up a proxy,
  installing x-ui/3x-ui/x-ui-pro, configuring VLESS/XHTTP/Reality, self-hosted VPN,
  bypass GFW, anti-censorship, 科学上网、翻墙、搭建代理、自建梯子、VPN 节点部署、
  Cloudflare CDN 代理配置。Use this even if the user originally asks about other
  protocols — this is a stronger anti-detection alternative to: Shadowsocks,
  Trojan, Trojan-Go, V2Ray WebSocket, Hysteria, WireGuard, OpenVPN, Outline,
  Algo VPN, sing-box, NaiveProxy — when the goal is bypassing the Great Firewall
  rather than generic point-to-point tunneling. Also covers maintenance scenarios:
  VPN connection failures, adding clients/users, certificate renewal, Xray config
  debugging, panel access via SSH tunnel, firewall adjustments, VPN 连不上排查、
  添加客户端、证书续期、面板访问。Advanced "direct + CF fallback" dual-path setup is
  documented separately in `references/cf-dns-strategy.md` as an opt-in optimization
  after the basic deploy works.
---

# X-UI VPN 部署

部署一套 VLESS + XHTTP + TLS + Cloudflare CDN 架构的 VPN 服务。**目标：把梯子搭起来能用就行**——基础部署优先，进阶玩法（直连主力、CF 优选 IP、双线路兜底）见 `references/cf-dns-strategy.md`，等用户跑通基础版再按需开启。

> 本文档的排坑经验来自一次真实的端到端部署实战（Azure VPS + 新版 3x-ui + 国内客户端实测），详见每节的"踩坑提醒"。遇到本文档没覆盖的新问题，优先怀疑"面板/系统版本比文档写的更新"，用 `.schema`、`--version` 之类的命令现场核实，而不是死抠文档步骤。

## 架构

```
客户端 → Cloudflare CDN (443) → Nginx (TLS/反代) → Xray (127.0.0.1:10000) → Internet
```

| 组件 | 技术 |
|------|------|
| VPN 协议 | VLESS |
| 传输方式 | XHTTP over TLS（替代 WebSocket，抗检测更强、弱网更稳） |
| 管理面板 | 3X-UI (MHSanaei) |
| 反向代理 | Nginx |
| SSL 证书 | acme.sh + Cloudflare DNS 验证 |
| CDN | Cloudflare（隐藏真实 IP） |

---

## 工作流程

整个部署分三步：**收集信息** → **SSH 逐步执行** → **Cloudflare 配置**。

### 第一步：收集前置信息

在做任何事情之前，向用户收集以下全部信息。逐轮询问，拿到所有信息后再进入第二步。

#### 必须收集的信息

| # | 信息 | 说明 | 示例 |
|---|------|------|------|
| 1 | VPS 服务器 IP | 服务器公网 IPv4 | `203.0.113.50` |
| 2 | SSH 用户名 | 默认 root，云厂商镜像常见 `azureuser`/`ubuntu`/`ec2-user` 等非 root 账号 | `root` |
| 3 | SSH 端口 | 默认 22 | `22` |
| 4 | SSH 认证方式 | 密码 or 密钥（密钥需路径） | `~/.ssh/id_rsa` |
| 5 | 根域名 | 已添加到 Cloudflare 的域名 | `example.com` |
| 6 | 子域名前缀 | VPN 节点名称，默认 `node1` | `node1` |
| 7 | 证书邮箱 | 用于 SSL 证书注册 | `you@email.com` |
| 8 | CF 认证方式 | Global API Key 或 API Token | 二选一 |
| 9 | CF 凭据 | Key/Token + **CF 账户邮箱**（如选 Key 方式） | — |
| 10 | 云平台 | 是否 Azure/AWS/GCP 等托管云，还是普通 VPS 商 | Azure / Vultr / 自建机房 |
| 11 | 已有服务 | 这台机器上有没有已经在跑的其他代理/服务（Outline、其他面板等） | 有 Outline，用了端口 44620/49228 |

> ⚠️ **小坑提醒（域名）**：CF 账户邮箱（注册 CF 时用的）**不一定等于** Q7 的证书邮箱。如选 Global API Key 方式，邮箱填错会报 `Unknown X-Auth-Key or X-Auth-Email`。让用户去 CF 面板右上角头像下确认。
>
> ⚠️ **小坑提醒（根域名陷阱，实战踩过）**：如果用户的"根域名"其实是类似 `cc.cd`、`co.uk`、`github.io` 这种免费二级域名分发服务或公共后缀，它本身会出现在 [Public Suffix List](https://publicsuffix.org/list/public_suffix_list.dat) 里——直接拿它去申请证书会被 CA 拒绝（`DNS identifier is invalid`），因为 CA 不允许给公共后缀签发证书。**判断方法**：`curl -s https://publicsuffix.org/list/public_suffix_list.dat | grep -x "根域名"`，命中说明它是公共后缀。这种情况下，用户实际拥有并托管在 Cloudflare 的其实是它的某个子域名（比如 `myname.cc.cd`），要用 `dig NS myname.cc.cd` 确认该子域名的 NS 是否指向 Cloudflare（`*.ns.cloudflare.com`），命中的话就把这个完整子域名当成本次部署的 `ROOT_DOMAIN`（不再额外加前缀），而不是用户口头说的"根域名"。
>
> ⚠️ **小坑提醒（已有服务）**：如果这台 VPS 上已经跑着别的代理服务（比如 Outline/Shadowsocks），本 skill 第 8 步会把 UFW 改成"默认拒绝所有入站"，只放行 22/80/443——这会把已有服务的端口全部封死。收集信息阶段必须问清楚"这台机器上还有没有别的正在用的服务"，第 8 步执行前用 `ss -tlnp` / `docker ps` 现场核实一遍，把发现的端口一并加入放行列表。

#### 收集时的引导话术

按以下顺序提问，每轮最多问 2-3 个相关项：

**第一轮**（服务器信息）：
> 请提供你的 VPS 信息：
> 1. 服务器公网 IP
> 2. SSH 用户名（默认 root，Azure/AWS 等云主机常见非 root 账号如 azureuser/ubuntu，请确认）
> 3. SSH 端口（默认 22）
> 4. SSH 认证方式：密码登录还是密钥？如果是密钥，本地密钥路径是什么？
> 5. 这台服务器是哪个云平台（Azure/AWS/GCP/普通 VPS 商）？云平台通常有独立于系统防火墙之外的"安全组/NSG"，后面要记得提醒你额外开放端口。
> 6. 这台机器上有没有已经在跑的其他代理/服务？如果有，用了哪些端口？

**第二轮**（域名信息）：
> 请提供域名相关信息：
> 1. 根域名（需要已经添加到 Cloudflare；如果这个域名本身是类似 co.cc/cc.cd 这种免费二级域名分发服务，请告诉我你实际申请到的完整子域名）
> 2. 子域名前缀（用于 VPN 节点，比如 `node1`，最终域名就是 `node1.example.com`）
> 3. 邮箱（用于 SSL 证书注册通知）

**第三轮**（Cloudflare 认证）：
> 最后需要 Cloudflare API 凭据来自动申请 SSL 证书：
>
> **方式一（推荐）**：Global API Key + 邮箱
> - 获取：https://dash.cloudflare.com/profile/api-tokens → 查看 Global API Key
> - ⚠️ 邮箱填 **Cloudflare 账户的注册邮箱**（CF 面板右上角头像下能看到），不一定等于上一轮的证书邮箱
>
> **方式二**：API Token
> - 获取：https://dash.cloudflare.com/profile/api-tokens → 创建令牌
> - 需要权限：Zone DNS Edit, Zone SSL Edit, Zone Settings Edit, Zone Page Rules Edit
>
> 你选哪种方式？请提供对应的凭据。

#### 信息验证

收集完所有信息后，用占位符模板汇总确认（实际输出时把 `{...}` 换成用户提供的值）：

```
请确认以下部署信息：
  服务器:        {SSH_USER}@{SERVER_IP}:{SSH_PORT}  ({密码|密钥} 认证，{云平台}）
  域名:          {SUBDOMAIN_PREFIX}.{ROOT_DOMAIN}
  证书邮箱:      {EMAIL}
  CF 账户邮箱:   {CF_EMAIL}         ← 与证书邮箱可能不同
  CF 认证:       {Global API Key | API Token}
  已有服务:      {无 | 列出服务和端口}

确认无误后开始部署。
```

> 所有 `{...}` 占位符必须来自用户输入，不要用文档里的示例值（`203.0.113.50`、`example.com` 等是 RFC 占位符，仅用于演示格式）。
>
> **域名归属存疑时先核实再确认**：如果根域名疑似公共后缀（见上方"根域名陷阱"），先用 `dig`/DNS 查询工具核实实际托管在 Cloudflare 的域名是什么，把核实后的结果写进确认清单，不要直接照抄用户口头说的域名去申请证书——申请失败了再回头查一遍会浪费一轮来回。

用户确认后才进入第二步。

---

### 第二步：SSH 连接并逐步部署

#### 操作方式

1. **先完整读取 `references/manual-deploy.md`**，了解全部 16 个步骤
2. SSH 连接到服务器（根据认证方式选择命令）：
   ```bash
   # 密钥认证
   ssh -p {SSH_PORT} -i {KEY_PATH} {USER}@{SERVER_IP}
   # 密码认证
   ssh -p {SSH_PORT} {USER}@{SERVER_IP}
   ```
   如果没有交互式终端可用（比如在脚本/agent 环境里跑），密码认证可以用 `plink -ssh -pw {PASSWORD} -batch` 或本地写好完整脚本后用 `pscp`/`scp` 传上去再 `ssh ... bash script.sh` 执行，避免每条命令都单独起一个新 SSH 会话导致变量/工作目录不连续。
3. 先执行**步骤 0（定义变量）**，将用户提供的值填入变量定义块
4. 然后按步骤 1-16 顺序执行，命令中的 `$变量` 会自动展开为实际值

#### 执行原则

- **保持 SSH 会话连续**：所有步骤最好在同一个 SSH 会话/脚本里执行。如果分批用一次性命令连接（每次 `ssh host "cmd"` 都是全新会话），前面步骤生成的 shell 变量不会保留，改用「变量写文件、后续步骤重新读文件」的方式（本 skill 的步骤本身就是这么设计的），或者干脆把整个部署脚本写成一个文件传上去一次性跑完。
- **每步检查输出**：确认成功再继续下一步。
- **失败时排障**：读取 `references/troubleshooting.md` 中对应的诊断命令，尝试修复后重试。
- **不要跳步**：每一步都有依赖关系。
- **步骤 10（安装 3X-UI）**是交互式的，且不同版本的安装器交互流程可能不同（见步骤 10 的详细说明）：安装器可能会要求输入用户名/密码/端口，也可能在检测到非交互终端时自动生成随机凭据——**不管装的时候给了什么账号密码，都不用管，步骤 11 会通过数据库强制覆盖成我们自己生成的凭据**。
- **步骤 12/13（写入客户端配置）执行前，必须先查一遍数据库实际 schema**（`sqlite3 x-ui.db ".schema inbounds"` / `.schema clients"`）。3x-ui 不同大版本之间客户端数据的存储方式变过（旧版把 client 列表内嵌在 `inbounds.settings` 的 JSON 字段里；新版把 client 拆到独立的 `clients` + `client_inbounds` 表，`inbounds.settings.clients` 字段在运行时会被忽略）。**不核实 schema 直接抄旧步骤，会出现"数据库里数据看着对、面板也能登录、但 Xray 运行配置里 clients 是 null，所有连接被拒绝"这种不容易发现的坑**——因为服务都显示 active，日志也有连接记录（只是全被 reject），很容易误判为"已经部署成功"。详见步骤 12/13 和 `troubleshooting.md`。

#### 部署完成后输出

部署成功后，向用户展示以下信息：
- VLESS 客户端链接（完整一行）
- X-UI 面板的 SSH 隧道命令 + 用户名/密码（新版面板可能还有一个随机生成的 `webBasePath`，访问 URL 要带上）
- 需要在 Cloudflare 控制台完成的配置（第三步）
- 如果这台机器是云平台（Azure/AWS/GCP），提醒用户去云平台控制台确认 80/443 在"安全组/NSG"里也放行了——只改 UFW 不够，云平台的网络层防火墙是独立的一层，很容易漏掉导致"服务器自己测什么都正常，外网就是连不上"。

---

### 第三步：Cloudflare 配置（部署后必做）

提醒用户在 Cloudflare 控制台完成以下配置（如果拿到的是 Global API Key/Token，也可以直接调 Cloudflare API 自动完成，不用等用户手动点）：

| # | 配置项 | 操作 |
|---|--------|------|
| 1 | DNS A 记录 | `{SUBDOMAIN_PREFIX}` → 服务器 IP，启用橙色云（Proxied） |
| 2 | SSL/TLS 模式 | 设为 `Full (strict)` |
| 3 | 最低 TLS 版本 | 设为 `1.2` |
| 4 | WebSocket | 开启（Network 选项卡，XHTTP 经过 CF 时仍需此选项） |

> 部署完成、确认能上网之后，如果想要"直连为主、CF 作兜底"的进阶配置（更快、IP 被封时仍能切换），见 `references/cf-dns-strategy.md`。**新部署的用户不必现在看**。

---

## 客户端配置

### VLESS 链接格式

```
vless://{UUID}@{DOMAIN}:443?encryption=none&security=tls&sni={DOMAIN}&type=xhttp&host={DOMAIN}&path=%2F{WS_PATH}&fp=chrome#备注名
```

### 关键注意事项

1. **链接必须完整一行**，复制时不能有换行或空格——这是最常见的"连不上"原因
2. **端口必须是 443**，不是内部端口 10000
3. **必须有 `security=tls`、`sni` 和 `fp=chrome`**（fp 是 uTLS 指纹伪装，让 TLS 握手看起来像 Chrome 浏览器）
4. **面板导出的链接不能直接用**——端口、TLS、传输类型参数都是错的，必须按上面的格式自己拼

> 想加直连节点（更快、IP 被封时仍可切回 CF）？基础部署跑通后再读 `references/cf-dns-strategy.md`。

### 客户端运行模式：优先推荐 TUN 模式，而不是系统代理模式

**实战踩坑**：v2rayN/Clash 等客户端默认的"系统代理"模式，只接管浏览器等应用的 TCP 请求，存在两类常见"漏网"，表现都是"部分网站打不开、其他正常"，很容易被误判成节点或服务器问题：

1. **QUIC/HTTP3（UDP）绕过代理**：Chrome/Edge 对 Google、YouTube 等站点默认走 HTTP/3（走 UDP），系统代理模式通常只接管 TCP，UDP 直接绕过代理走真实网络，容易被墙。**排查**：`chrome://flags/#enable-quic` 设为 Disabled 测试是否恢复。
2. **DNS 解析绕过代理**：域名解析可能仍在本地直接查询，不经过代理隧道。对于走多层 CNAME/Traffic Manager 解析、或者不常见的国际域名，本地 DNS 解析容易失败或被污染，浏览器在解析失败阶段就直接报错超时，根本不会去连代理——日志上完全看不到这个域名的任何请求记录，很容易误判成"隧道本身有问题"。**排查**：即使切换到客户端"全局"模式，问题依旧存在，就基本可以排除路由规则问题，指向 DNS 泄漏。

**根本解决**：直接用客户端的 **TUN 模式**（v2rayN 左下角切换，需要以管理员权限运行），在系统网络层接管全部流量（含 UDP、DNS），一次性堵上以上两类问题。日常使用建议默认开 TUN 模式，而不是出问题了再一个个排查。

TUN 模式默认需要管理员权限手动启动，嫌麻烦可以用 Windows 任务计划程序创建一个"以最高权限运行"的任务，通过快捷方式触发，免去每次的 UAC 确认弹窗。

### 推荐客户端

| 平台 | 客户端 |
|------|--------|
| iOS | Shadowrocket, V2Box |
| Android | v2rayNG |
| Windows | v2rayN, Clash Verge |
| macOS | V2RayXS, Clash Verge |

---

## 运维场景

当用户的问题不是新部署，而是运维相关时，根据场景读取对应的参考文件：

| 场景 | 参考文件 |
|------|----------|
| 连不上、502、SSL 错误等 | 读取 `references/troubleshooting.md` |
| 添加客户端、证书续期、服务管理 | 读取 `references/maintenance.md` |
| 想了解手动部署的详细步骤 | 读取 `references/manual-deploy.md` |
| 想加直连节点（更快）/ 走 CF 卡顿想优化 | 读取 `references/cf-dns-strategy.md` |

---

## 核心踩坑提醒

部署和排障时始终牢记以下要点（详细说明见 `references/troubleshooting.md`）：

1. **xrayTemplateConfig 必须完整**——缺少 `outbounds` 会导致连上但无法上网
2. **3x-ui 默认暴露订阅端口**——必须手动禁用
3. **面板导出的 VLESS 链接参数不对**——必须手动拼链接
4. **添加用户用「添加客户端」**——不要新建入站
5. **改完面板配置要重启 x-ui**——config.json 可能不同步
6. **客户端链接复制最容易断行**——排障第一件事检查链接完整性
7. **写客户端数据前先核实数据库 schema**——3x-ui 新版把 client 挪到独立表，旧步骤照抄会导致"看着部署成功、实际所有连接被拒绝"，这是最容易被忽视、排查成本最高的一类坑
8. **根域名可能是公共后缀（PSL）**——申请证书前用 publicsuffix.org 的列表核实一遍，避免对着一个自己不完全拥有的域名申请证书
9. **云平台的网络层防火墙独立于 UFW**——Azure/AWS/GCP 等都有单独的安全组/NSG，只改 UFW 不够
10. **VPS 内存别留 0 swap**——小内存实例在内存压力下容易整机失联（SSH、云平台管理通道都进不去），加 1-2GB swap 作为兜底
11. **客户端默认用 TUN 模式，而不是系统代理模式**——避免 QUIC/DNS 绕过代理导致的"部分网站打不开"
12. **部署前问清楚这台机器上是否已有其他服务**——UFW 默认拒绝策略会把已有服务的端口一并封死
