# Changelog

## 2026-09-09 — AlmaLinux 部署 + 3x-ui 3.7.0 适配

基于第二次端到端部署实战（AlmaLinux 9.7 VPS + 3x-ui 3.7.0 + Windows 本机 plink 非交互执行 + Cloudflare API 自动配置）补充：

### 修复

- **`manual-deploy.md` 步骤 13**：3x-ui 3.7.0 上直接往 `inbounds` 表 INSERT 的行会被 config 生成逻辑整个忽略（`config.json` 的 inbounds 为空数组、10000 不监听、日志无任何报错），比 7 月记录的"clients 为 null"更彻底。新增"路线 A：走面板 API"作为新版 schema 的首选，用 `x-ui setting -getApiToken` 的 Bearer token 调 `/panel/api/inbounds/add`，原手写 sqlite 方式降级为"路线 B 兜底"。附带三个细节：登录接口对 curl 返回 403（CSRF），client JSON 里 `tgId` 必须是 int（去掉即可），API 建的 tag 是 `in-10000-tcp`（相关查询改为按 `port=10000`）。
- **`manual-deploy.md` 步骤 10/11**：3.7+ 安装器有正式的 `NONINTERACTIVE=1` + `XUI_*` 环境变量模式，直接喂我们生成的凭据；面板配置改用 `x-ui setting -username/-password/-port/-webBasePath/-listenIP` 一条命令，不再需要 bcrypt + 改 users 表。订阅开关等仍走 sqlite，但改成"先查再 INSERT/UPDATE"的 `set_setting` 写法，根治 `INSERT OR REPLACE` 重复行。
- **`manual-deploy.md` 步骤 4**：acme.sh 默认 CA 改为 Let's Encrypt；`--reloadcmd` 加 `|| true`（此时 nginx 还没起）；证书目录参数化为 `CERT_DIR`。

### 新增

- **RHEL 系支持**（AlmaLinux/Rocky/CentOS Stream）：步骤 1 新增发行版检测和"差异对照表"，步骤 2/6/7/8/9/14/16 各加 RHEL 小节——`dnf` + EPEL、`firewalld` 替代 ufw、fail2ban 用 `firewallcmd-rich-rules` + `backend = systemd`、整体重写 `nginx.conf`（自带的没有 `ssl_protocols`/`server_tokens` 行，原 `sed` 静默不生效）、站点写 `/etc/nginx/conf.d/`、证书放 `/etc/nginx/cert/`、SELinux Enforcing 时 `setsebool httpd_can_network_connect`。原则：不让用户重装系统。
- 非交互 SSH 执行模式：步骤 0 新增 `/root/.secrets/deploy-vars.sh` 变量文件约定，每步脚本 `source` 后执行，适配 `plink -batch ... "bash -s" < step.sh` 逐步喂脚本的方式。
- 步骤 14/15 健康检查补充：公网监听面核对、acme cron 是否存在、XHTTP 路径本机和经 CF 两处 curl（期望 404）、CF 响应头核对。
- Cloudflare 新格式 Global API Key（`cfk_` 前缀）说明：与旧 key 一样走 `CF_Key`+`CF_Email`，部署前先本地 curl 一次 zones 接口验证。
- `troubleshooting.md` 新增：3.7.0 入站被忽略的变体及 API 修复、RHEL 系常见报错（ufw 找不到、fail2ban 找不到 auth.log、nginx sed 不生效、SELinux 拒绝反代）。
- `maintenance.md` 新增：API token 加客户端的方式、RHEL 系路径/命令对照。

## 2026-07-02 — 实战复盘更新

基于一次完整的端到端部署实战（Azure VPS + 新版 3x-ui + 国内客户端实测）补充的经验和修复：

### 修复

- **`manual-deploy.md` 步骤 13（严重 bug）**：修正客户端数据写入逻辑。新版 3x-ui 把客户端数据从 `inbounds.settings` 内嵌 JSON 挪到了独立的 `clients` + `client_inbounds` 表，旧版步骤照抄会导致所有服务显示正常、但 Xray 运行配置里 `clients` 为 `null`，所有连接被静默拒绝——这是本次实战中最隐蔽、排查成本最高的一个问题。现在步骤 13 会先检测 schema 版本，再选择对应的写入方式，并新增了"验证 clients 真的非空"的强制检查步骤。
- **`manual-deploy.md` 步骤 11**：修正 `INSERT OR REPLACE` 在无 UNIQUE 约束的新版 `settings` 表上会产生重复行的问题，补充检测和清理命令。

### 新增

- 域名归属核实流程（步骤 0）：部署前检测根域名是否为 Public Suffix List 上的公共后缀，避免对着一个自己不完全拥有的域名申请证书导致 `DNS identifier is invalid` 报错。
- Swap 安全垫（步骤 2.5）：为内存较小的 VPS 加 swap，防止资源压力下整机失联（连 SSH、云平台管理通道都进不去）。
- 已有服务检测（步骤 1 + 步骤 8）：部署前摸清服务器上是否已有其他代理服务（如 Outline），避免 UFW 默认拒绝策略把已有服务端口一并封死。
- 云平台安全组/NSG 提醒（步骤 8 + troubleshooting）：UFW 只是系统级防火墙，Azure/AWS/GCP 等云平台还有独立的网络级防火墙，容易漏配。
- Cloudflare API 自动配置（新增步骤 15）：有 API 凭据时直接调 API 完成 DNS/TLS 设置，不用等用户手动点。
- 客户端 TUN 模式建议：记录了 QUIC/UDP 绕过代理、DNS 解析绕过代理导致"个别网站打不开"的两类真实案例及排查方法，建议默认用 TUN 模式而不是系统代理模式。
- `troubleshooting.md` 新增章节：clients 为 null、根域名是公共后缀、云平台安全组未放行、个别网站打不开、已有服务被断、服务器整体失联（内存耗尽）。
- `troubleshooting.md` 新增"临时开启 Xray 详细访问日志"操作指南，含排障完成后必须关闭的提醒。

### 说明

以上问题均来自同一次真实部署会话中依次遇到并解决的实际故障，不是假设性场景。
