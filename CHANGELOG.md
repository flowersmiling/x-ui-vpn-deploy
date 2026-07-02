# Changelog

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
