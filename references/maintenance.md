# 日常维护

## 服务管理

```bash
# 查看所有服务状态
systemctl is-active nginx x-ui fail2ban

# 重启服务
systemctl restart nginx
systemctl restart x-ui

# 查看日志
journalctl -u x-ui --since "1 hour ago"
tail -30 /var/log/nginx/error.log

# 防火墙（按发行版二选一）
ufw status                  # Debian/Ubuntu
firewall-cmd --list-all     # RHEL 系（AlmaLinux/Rocky/CentOS）
```

> 先 `cat /etc/os-release` 确认是哪个发行版再动手：RHEL 系的包管理是 `dnf`、防火墙是 `firewalld`、Nginx 站点在 `/etc/nginx/conf.d/`、证书在 `/etc/nginx/cert/`（详见 `manual-deploy.md` 步骤 1 的差异对照表）。

---

## 证书管理

```bash
# 查看证书有效期（RHEL 系部署的证书目录是 /etc/nginx/cert/）
openssl x509 -in /root/cert/fullchain.cer -noout -enddate

# 查看 acme.sh 定时任务（自动续期）
crontab -l | grep acme

# 手动强制续期
~/.acme.sh/acme.sh --renew -d $(ls ~/.acme.sh/ | grep _ecc | head -1 | sed 's/_ecc//') --force --ecc
```

证书由 acme.sh 自动续期（每 60 天），续期后自动 reload Nginx。正常情况下不需要手动操作。

---

## 添加新客户端

**不要新建入站！** 在现有入站里「添加客户端」：

1. 通过 SSH 隧道进面板：
   ```bash
   ssh -L 54321:localhost:54321 user@SERVER_IP
   # 浏览器访问 http://localhost:54321（新版面板可能带一段随机 webBasePath 前缀，从数据库查：
   #   sqlite3 /etc/x-ui/x-ui.db "SELECT value FROM settings WHERE key='webBasePath';"）
   ```
2. 入站列表 → 点 `+` 展开客户端列表
3. 点「添加客户端」→ 填备注 → UUID 自动生成 → 保存
4. 用以下模板拼链接发给用户：
   ```
   vless://<新UUID>@DOMAIN:443?encryption=none&security=tls&sni=DOMAIN&type=xhttp&host=DOMAIN&path=%2FWS_PATH&fp=chrome#用户备注
   ```

> 新建入站会导致：绕过 Nginx 伪装、端口暴露公网、无 TLS 保护。

不想开隧道进面板的话，3.7+ 可以直接在服务器上用 API token 加客户端（和部署时创建入站是同一套鉴权，token 在 `/root/.secrets/xui_api_token.txt`，或 `x-ui setting -getApiToken` 现查）：

```bash
TOKEN=$(cat /root/.secrets/xui_api_token.txt)
XUI_PORT=$(sqlite3 /etc/x-ui/x-ui.db "SELECT value FROM settings WHERE key='webPort' LIMIT 1;")
INBOUND_ID=$(sqlite3 /etc/x-ui/x-ui.db "SELECT id FROM inbounds WHERE port=10000;")
NEW_UUID=$(cat /proc/sys/kernel/random/uuid)
# settings 是 JSON 字符串；client 里不要带 tgId（3.7 改成 int 了）
python3 - "$INBOUND_ID" "$NEW_UUID" > /tmp/client.json <<'PY'
import json, sys
settings = json.dumps({"clients":[{"id":sys.argv[2],"flow":"","email":"user2","limitIp":0,"totalGB":0,"expiryTime":0,"enable":True,"subId":"","reset":0}]})
print(json.dumps({"id": int(sys.argv[1]), "settings": settings}))
PY
curl -s -X POST "http://127.0.0.1:$XUI_PORT/panel/api/inbounds/addClient" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" --data @/tmp/client.json
systemctl restart x-ui
echo "$NEW_UUID"
```

> `addClient` 这个接口是 3x-ui 官方 API 列表里的，字段格式和 `inbounds/add` 一致；部署实战里验证过的是 `inbounds/add`，`addClient` 加完务必按下一节的方法确认 `config.json` 里真的多了这个 UUID。

### 通过面板加完客户端后，务必验证真的生效了

新版 3x-ui 客户端数据存在独立的 `clients` + `client_inbounds` 表（详见 `troubleshooting.md` "clients 为 null"一节）。通过面板界面添加通常没问题（面板自己知道该写哪张表），但如果是手动改数据库加的客户端，一定要重启后确认运行配置里能看到：

```bash
systemctl restart x-ui
sleep 2
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
for ib in cfg.get('inbounds', []):
    print(ib.get('tag'), json.dumps(ib.get('settings')))
"
```

---

## 面板修改后验证

面板修改入站后，Xray 运行配置可能不同步：

```bash
# 检查 config.json 是否正确
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('Outbounds:', cfg.get('outbounds'))
print('DNS:', cfg.get('dns'))
for ib in cfg.get('inbounds', []):
    print(ib.get('tag'), 'clients:', ib.get('settings', {}).get('clients'))
"

# 不确定时直接重启
systemctl restart x-ui
```

---

## 安全加固清单

### 已由部署流程配置

- [x] 防火墙（Debian 用 UFW，RHEL 系用 firewalld）仅开放 SSH/80/443（+ 部署时发现的已有服务端口）
- [x] X-UI 面板仅 localhost 访问
- [x] 订阅端口已禁用
- [x] Fail2Ban 防暴力破解（SSH 3 次失败封 2 小时）
- [x] Nginx 隐藏版本号
- [x] TLSv1/TLSv1.1 已禁用
- [x] HSTS / X-Frame-Options / X-Content-Type-Options 安全头部
- [x] Cloudflare 隐藏真实 IP
- [x] Swap 内存安全垫（防止资源耗尽导致整机失联）

### 建议额外配置

- [ ] SSH 密钥认证（禁用密码登录）
- [ ] 定期更换 XHTTP 路径
- [ ] 定期更换 Cloudflare API Token
- [ ] 如果 VPS 云平台有独立安全组/NSG，定期复核里面的规则和 UFW 是否一致

### 敏感文件权限

```bash
chmod 700 /root/.secrets/
chmod 600 /root/.secrets/*.txt
chmod 600 /root/vpn-config.txt
```

---

## 配置文件位置速查

| 文件 | 路径 |
|------|------|
| 密钥目录 | `/root/.secrets/` |
| XHTTP 路径 | `/root/.secrets/ws_path.txt` |
| 客户端 UUID | `/root/.secrets/vless_uuid.txt` |
| X-UI 凭据 | `/root/.secrets/xui_username.txt` / `xui_password.txt` |
| X-UI API token（3.7+） | `/root/.secrets/xui_api_token.txt`，或 `x-ui setting -getApiToken` 现查 |
| 部署变量文件（非交互部署时） | `/root/.secrets/deploy-vars.sh`，含域名、端口、CERT_DIR、CF 凭据 |
| 伪装站公司名 | `/root/.secrets/decoy_name.txt`（部署时随机生成的英文名，改了要同步改 `/var/www/$DOMAIN/index.html`）|
| Nginx 全局 | `/etc/nginx/nginx.conf` |
| Nginx 站点 | `/etc/nginx/sites-available/<域名>`（RHEL 系：`/etc/nginx/conf.d/<域名>.conf`） |
| Fail2Ban | `/etc/fail2ban/jail.local` |
| X-UI 数据库 | `/etc/x-ui/x-ui.db` |
| Xray 运行配置 | `/usr/local/x-ui/bin/config.json`（自动生成，勿手动改） |
| Xray 详细访问日志（排障临时开启） | `/var/log/x-ui/xray-access.log`（实际路径以运行配置里的 `log.access` 为准，不同版本可能不同） |
| SSL 证书 | `/root/cert/`（RHEL 系：`/etc/nginx/cert/`；以 `deploy-vars.sh` 里的 `CERT_DIR` 为准） |
| acme.sh | `~/.acme.sh/<根域名>_ecc/` |
| 伪装网站 | `/var/www/<域名>/index.html` |
| 配置汇总 | `/root/vpn-config.txt` |
| Swap 文件 | `/swapfile` |

---

## 定期维护计划

### 每月

- [ ] 检查 SSL 证书有效期
- [ ] 查看 Fail2Ban 拦截记录：`fail2ban-client status sshd`
- [ ] 更新系统软件包：`apt update && apt upgrade`（RHEL 系：`dnf upgrade`）
- [ ] 看一眼 VPS 控制台的当月流量用量（KiwiVM/Vultr/GCP 等首页通常都有），接近上限时准备应对
- [ ] `free -h` 看一眼内存/swap 使用情况，`df -h /` 看一眼磁盘，尤其是小内存实例——排障时临时开的详细日志有没有忘记关（`grep -c . /var/log/x-ui/xray-access.log`，正常情况这个文件应该很小或不存在）

### 每季度

- [ ] 更换 XHTTP 路径（同步更新 Nginx + X-UI + 客户端）
- [ ] 备份配置文件
- [ ] 审查防火墙规则（UFW/firewalld + 云平台安全组/NSG 两边都要看）
- [ ] 检查 3x-ui 面板版本，如果做过大版本升级，留意数据库 schema 是否变化（`.schema inbounds` / `.schema clients` 和记录里的对不对得上）

### 每年

- [ ] 更换 Cloudflare API Token
- [ ] 审查 X-UI 用户列表
- [ ] 检查 Xray 版本更新和弃用通知
