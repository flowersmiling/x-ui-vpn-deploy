# x-ui-deploy

一个 [Claude Code](https://claude.com/claude-code) Skill：在一台全新 VPS 上，从零部署一套 **VLESS + XHTTP + TLS + Cloudflare CDN** 架构的自建代理服务（基于 [3X-UI](https://github.com/MHSanaei/3x-ui) 面板），目标是"把梯子搭起来能用就行"，同时覆盖日常运维、故障排查和进阶双线路优化。

本仓库的内容脱胎于一次真实的端到端部署实战（Azure VPS + 新版 3x-ui + 国内客户端实测），踩过的坑都已经写进对应文档，不是纸上谈兵的操作手册。

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

## 开始前准备

在让 Claude Code 执行部署之前，先备齐这些东西，能少走很多回头路：

| # | 需要什么 | 说明 |
|---|---------|------|
| 1 | 一台全新 VPS | Debian/Ubuntu 为主线；AlmaLinux/Rocky/CentOS Stream 等 RHEL 系也支持（dnf/firewalld/conf.d 的差异已写进 `manual-deploy.md`，不用重装）。能 SSH 登录（root 或有 sudo 权限的账号均可）。云平台（Azure/AWS/GCP 等）也可以，但要留意它们在系统防火墙之外还有一层独立的安全组/NSG，见下方"已知的坑" |
| 2 | 已经解析到 Cloudflare 的域名 | 域名（或至少一个子域名）的 NS 记录已指向 Cloudflare。**注意**：如果你的"域名"其实是 `cc.cd`/`co.cc` 这类免费二级域名分发服务，本身可能在 [Public Suffix List](https://publicsuffix.org/list/public_suffix_list.dat) 上，不能直接拿来签证书，需要用你实际申请到的完整子域名，详见 troubleshooting.md |
| 3 | Cloudflare API 凭据 | Global API Key + 账户注册邮箱，或者一个有 Zone DNS/SSL/Settings 编辑权限的 API Token，二选一，用于自动申请 SSL 证书和配置 DNS |
| 4 | 一个用于证书注册的邮箱 | 不需要和 Cloudflare 账户邮箱是同一个 |
| 5 | 已安装 [Claude Code](https://claude.com/claude-code) | 本项目是给它用的 Skill，不是独立脚本 |
| 6 | （可选）这台 VPS 上是否已有其他服务在跑 | 如果有（比如已经部署了 Outline），提前记下它们用的端口，避免部署过程中被防火墙规则误封 |

准备齐这些信息后，跟 Claude Code 说"帮我部署一个 VPN"，它会按 `SKILL.md` 里的流程逐项向你确认，再开始执行。

## 这是什么、怎么用

这不是一个可以直接运行的安装脚本，而是一套写给 [Claude Code](https://claude.com/claude-code) 的 **Skill**——把它放进 `~/.claude/skills/x-ui-deploy/` 之后，跟 Claude Code 说"帮我部署一个 VPN"之类的话，它会读取这套文档、通过 SSH 连接你的服务器、逐步完成部署，并在遇到问题时按文档里记录的踩坑经验自主排查。

### 安装

```bash
git clone https://github.com/flowersmiling/x-ui-vpn-deploy.git ~/.claude/skills/x-ui-deploy
```

（Windows 上是 `%USERPROFILE%\.claude\skills\x-ui-deploy`）

安装完之后，直接跟 Claude Code 说"帮我部署一个 VPN / 搭个代理"即可触发。也可以手动把里面的命令一步步跑，当纯粹的部署 checklist 用。

### 目录结构

```
SKILL.md                        # 主入口：信息收集流程、整体决策逻辑、核心踩坑提醒
references/
  manual-deploy.md              # 16 步完整部署蓝图，每步都是可直接执行的命令
  troubleshooting.md            # 故障排查手册，按症状分类，附确诊命令和修复步骤
  maintenance.md                # 日常运维：加客户端、续证书、安全加固清单
  cf-dns-strategy.md            # 进阶：直连主力 + Cloudflare 兜底的双线路设计
```

## 这套方案解决了什么问题

- **抗检测更强**：用 XHTTP（Xray 官方推荐替代 WebSocket 的传输方式）而不是更容易被识别的旧协议，流量特征更像正常网页浏览
- **隐藏真实 IP**：客户端连的是 Cloudflare 边缘节点，真实服务器 IP 不暴露，即使 IP 被封也能通过 CF 兜底
- **伪装站点防指纹**：每次部署自动生成不同的伪装公司站点内容，避免批量部署的服务器共享同一特征被识别
- **面板不暴露公网**：3X-UI 管理面板仅监听 localhost，只能通过 SSH 隧道访问
- **文档来自真实踩坑**，覆盖了域名归属陷阱、面板版本 schema 变化、云平台防火墙分层、客户端 UDP/DNS 绕过代理、服务器资源耗尽等一系列实战中真实遇到、容易被误诊断的问题

## 已知的几类容易踩的坑（详见 troubleshooting.md）

1. **根域名本身是公共后缀（Public Suffix List）**——比如 `cc.cd`、`co.cc` 这类免费二级域名分发服务，直接拿它申请证书会被 CA 拒绝
2. **3x-ui 面板版本差异导致的 schema 变化**——较新版本把客户端数据从内嵌 JSON 挪到独立数据库表，按旧文档操作会出现"服务全部 active、但所有连接被静默拒绝"这种极难发现的问题
3. **云平台的网络层防火墙独立于系统防火墙**——Azure/AWS/GCP 都有单独的安全组/NSG，只改 UFW 不够
4. **客户端"系统代理"模式漏掉 UDP/DNS**——导致 Google/YouTube 等站点或某些 SSO 登录页打不开，很容易被误判成服务器或线路问题
5. **小内存 VPS 没有 swap，资源压力下整机失联**——连 SSH 和云平台管理通道都进不去，只能靠串行控制台或重启恢复
6. **3x-ui 3.7.0 起手写 sqlite 插入的入站会被整个忽略**——`config.json` 里 inbounds 为空、端口不监听、日志无报错。新版走面板 API（Bearer token）创建入站，登录接口对 curl 是 403，`tgId` 字段改成了整数
7. **VPS 是 AlmaLinux/Rocky 这类 RHEL 系**——没有 ufw、没有 sites-available、自带 nginx.conf 结构不同、fail2ban 日志在 journald，按对照表换命令即可，不用重装成 Debian

## License

MIT
